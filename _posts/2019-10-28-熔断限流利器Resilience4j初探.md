---
layout:     post
title:      "熔断限流利器Resilience4j初探"
subtitle:   "Resilience4j Usage"
date:       2019-10-28
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Design
    - Resilience4j
    - RateLimit
---
> Resilience4j是一个轻量级的Java服务容错框架，它仅依赖Vavr，后者是基于Java8的大量使用函数式范式的框架。
>
> 本文会对其提供的限流和熔断方法进行一定的分析和测试。

### Vavr简介
比如我们有一个微服务的接口：
```java
// Simulates a microservice for user management
public interface UserService {
    Picture fetchProfilePicture(String userId);
};
```
它的一个调用方调用代码如下：
```java
try {
    profilePicture = userService.fetchProfilePicture(userId);
} catch(Exception e) {
    Logger.error("The world is not a perfect place ", e);
}
```
为了处理可能的异常错误，使用了 `try-catch` ，但该方式会让「错误处理逻辑」侵入到「业务逻辑」中，造成业务逻辑复杂，进而可能引入更多的bug。

所以，比较好的实现是能将业务逻辑和错误处理分离，简化业务逻辑代码：
```java
Supplier<Picture> fetchTargetPicture = () -> userService.fetchProfilePicture(targetID);
// in case of failure you'll receive some stub picture
Picture profilePicture = Try.ofSupplier(fetchTargetPicture)
    .recover(throwable -> Picture.defaultForProfile()).get();
```
上述代码使用了 `Try` ，它是 `Vavr` 提供的概念，通过 `recover` 进行错误处理，无需再自己捕获异常来处理，方法简洁，使用方便。

`Vavr` 更多的功能这里先不做介绍，重点是 `Resilience4j` 。

### Resilience4j功能
`Resilience4j` 提供了 `RateLimiter` ，`CircuitBreaker` ，`BulkHead` ，`TimeLimiter` ，`Retry` ，`Cache` 等功能。
其中RateLimiter 是一个限速器，CircuitBreaker 实现了断路器模式，BulkHead  限制并发，TimeLimiter实际上是一个超时器，
Retry 实现自动重试功能，Cache自动使用缓存。Resilience4j 使用 **装饰器模式** 实现容错功能，使用者只需在业务上层做一下包装即可，
无需侵入业务代码。

在使用上述这些功能一般分4步：

- 第一步：创建容错相关的配置

- 第二步：创建容错器

- 第三步：创建装饰器，用容错器去装饰业务逻辑

- 第四步：调用装饰器，调用业务逻辑
#### RateLimiter
`RateLimiter` （注意和 `Guava` 的实现区分）有3个配置，详细见以下代码的注释，可以理解为一个阻塞式的限速器，可以配置最大等待时间。
```java
import java.time.Duration;
import java.util.function.Supplier;
 
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import io.vavr.control.Try;
 
public class DemoRateLimiter {
    public static void main(String[] args) {
        // 1: 创建配置
        RateLimiterConfig config = RateLimiterConfig.custom()
                .timeoutDuration(Duration.ofMillis(2))  // 等待 token 的超时时间为 2 毫秒
                .limitRefreshPeriod(Duration.ofMillis(10))  // 每 10 毫秒生成1次 token
                .limitForPeriod(1)  // 每次生成 1 个token
                .build();
 
        // 2: 创建 RateLimiter
        RateLimiter rateLimiter = RateLimiter.of("backendName", config);
 
        // 3: 创建装饰器
        Supplier<String> restrictedSupplier = RateLimiter
                .decorateSupplier(rateLimiter, () -> {
                    return "Success"; // 业务逻辑
                });
 
 
        // 4: 发起业务调用
        for (int i = 0; i < 10; i++) {
            Try<String> result = Try.ofSupplier(restrictedSupplier)
                    .recover(throwable -> {
                        return "Failed";  // Fallback
                    });
            System.out.println(i + ": " + result.get());
        }
    }
}
```
输出
```
0: Success
1: Failed
2: Failed
3: Success
4: Failed
5: Failed
6: Failed
7: Success
8: Failed
9: Failed
```
可以看出最多等 2ms 没有 `token` 生成则直接执行 `recover` 逻辑，如果改为 20ms 等待则全部为 `Success` 。

#### BulkHead
`BulkHead` 可以实现「限制并发」功能，当发送请求的时候将并发数加1，当请求完成时将并发数减1，并且限制最大并发数。
这一点和使用 `Semaphore` 实现类似，都是用来控制并发数。
```java
import java.time.Duration;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;
import io.vavr.control.Try;
 
public class DemoBulkHead {
    public static void main(String[] args) {
        // 1：创建配置
        BulkheadConfig bulkheadConfig = BulkheadConfig.custom()
                .maxConcurrentCalls(1)  // 限制最大并发数为1
                .maxWaitDuration(Duration.ofMillis(1))// 当并发数超过阈值时的最大等待时间
                .build();
 
        // 2：创建 BulkHead
        Bulkhead bulkhead = Bulkhead.of("BulkHead", bulkheadConfig);
         
        // 3：创建装饰器
        Supplier<String> bulkheadSupplier = Bulkhead.decorateSupplier(bulkhead, () -> {return "Success";});
 
        // 4：发起调用
        for (int i = 0; i < 100; i++) {
            AtomicReference<Integer> atomicReference = new AtomicReference<>(i);
            // 使用多线程模拟并发场景
            new Thread(() -> {
                Try<String> result = Try.ofSupplier(bulkheadSupplier)
                        .recover(throwable -> {
                            return "Failed";
                        });
                System.out.println(atomicReference.get() + ": " + result.get());
            }).start();
        }
    }
}
```
注意测试线程数较少时可能无法得到预期的效果。

#### Retry
`Retry` 的使用相对简单一点儿，可以配置最大重试次数、重试间隔，也可以定义重试间隔的更新函数，比如让重试间隔按指数增长。
```java
import io.github.resilience4j.retry.IntervalFunction;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.vavr.CheckedFunction0;
import io.vavr.control.Try;
 
public class DemoRetry {
    public static void main(String[] args) {
        // 1：创建配置
        RetryConfig config = RetryConfig.custom()
                .maxAttempts(8) // 最大重试次数
                // .waitDuration(Duration.ofMillis(100)) // 设置重试间隔
                // 设置重试间隔100ms，并按指数增长，底数为1.5
                .intervalFunction(IntervalFunction.ofExponentialBackoff(100, 1.5))  
                .build();

        // 2：创建 Retry
        Retry retry = Retry.of("retry", config);

        AtomicInteger atomicInteger = new AtomicInteger(1);
        // 3：创建装饰器
        CheckedFunction0<String> retryableSupplier = Retry.decorateCheckedSupplier(retry, () -> {
            if (atomicInteger.getAndIncrement() < 10) { // 调整这里
                System.out.println("err");
                throw new IllegalArgumentException();
            }
            System.out.println("ok");
            return "Success";
        });

        // 4：发起调用
        Try<String> result = Try.of(retryableSupplier).recover((throwable) -> "Failed");
        System.out.println(result.get()); 
    }
}
```
通过测试可以发现我们模拟了前几次的失败，`Retry` 会自动根据配置帮我们重试，直到达到了 `maxAttempts`，同时可以观察到
重试的时间间隔是受 `intervalFunction` 函数控制的，这里是按照指数增长。

#### CircuitBreaker
`CircuitBreaker` 实际上是一个「状态机」，内部有3种状态：`Closed` ，`Open` ，`Half-Open` 。默认处于 `Closed` 状态，允许通过所有请求，
一旦请求的失败率达到某个阈值时，断路器就会变成 `Open` 状态，之后所有请求都会失败，经过一段时间后，断路器会进入 `Half-Open` 状态，
允许所有请求通过，再根据失败率判断进入 `Closed` 还是 `Open` 状态。
![state_machine](https://files.readme.io/39cdd54-state_machine.jpg)

内部的状态机类 `CircuitBreakerStateMachine` 使用了 `AtomicReference` 来保存状态，实现原子状态切换：
```java
private final AtomicReference<CircuitBreakerState> stateReference;

public void transitionToClosedState() {// 转换到Closed状态
    // 原子操作
    CircuitBreakerState previousState = stateReference.getAndUpdate(currentState -> {
        if (currentState.getState() == CLOSED) {
            return currentState;
        }
        return new ClosedState(this, currentState.getMetrics());
    });
    if (previousState.getState() != CLOSED) {
        publishStateTransitionEvent(
            StateTransition.transitionToClosedState(previousState.getState())
        );
    }
}
```

`CircuitBreaker` 为了统计过去一段时间请求的失败率，使用了2个 
`RingBuffer`（实际上是 `Bitset` ）来记录每次请求是成功还是失败。一个在 `Closed` 状态下使用，另一个在 `Half-Open` 状态下使用。
这里为什么使用了2个 `RingBuffer` 呢，因为 `CircuitBreaker` 只有在 `RingBuffer` 填满之后才会计算失败率，可以给 `RingBuffer`  
size配置不同的大小，比如让 `Half-Open` 下 `RingBuffer` 的size小一些，让处于 `Half-Open` 下的时间更短，判决的更快。

```java
import java.time.Duration;
import java.util.Random;
import java.util.function.Supplier;
 
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.vavr.control.Try;
 
public class DemoCircuitBreaker {
    public static void main(String[] args) {
        // 1： 创建配置
        CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
                .failureRateThreshold(50)  // 配置失败率阈值
                .waitDurationInOpenState(Duration.ofSeconds(60))  // 配置断路器在Open状态持续的时间，默认60秒之后会进入Half-Open状态。
                .ringBufferSizeInHalfOpenState(2)  // Half-Open状态下RingBuffer的大小
                .ringBufferSizeInClosedState(20)  // Closed状态下RingBuffer的大小
                .build();
 
        // 2：创建 CircuitBreaker
        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(circuitBreakerConfig);
        CircuitBreaker circuitBreaker = registry.circuitBreaker("testName");
 
        // 3： 创建装饰器
        Supplier<String> supplier = CircuitBreaker
                .decorateSupplier(circuitBreaker, () -> {
                    int v = new Random().nextInt(2);
                    if (v != 0) {
                        try {
                            Thread.sleep(100);
                            return "Good " + v;
                        } catch (Exception e) {
                            throw new IllegalArgumentException("Sleep Exception");
                        }
                    } else {
                        throw new IllegalArgumentException("Rand Exception");
                    }
                });
 
        // 3： 发起调用
        for (int i = 0; i < 100000; i++) {
            String result = Try.ofSupplier(supplier)
                    .recover(throwable -> "Bad").get();
            System.out.println(result);
        }
    }
}
```
`CircuitBreakerConfig` 内部基于 `ConcurrentHashMap` 实现，所以可以作为一个全局变量来维护 `CircuitBreaker` 实例。
```java
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
``` 

但需要注意的是 `CircuitBreaker` 并不会限制并发数，并发数的限制应该使用 `BulkHead` 。
> If 20 concurrent threads ask for the permission to execute a function and the state of the CircuitBreaker is closed, all threads are allowed to invoke the function. Even if the Ring Bit Buffer size is 15. The size of the Ring Bit Buffer does not mean that only 15 calls are allowed to run concurrently. If you want to restrict the number of concurrent threads, please use a Bulkhead. You can combine a Bulkhead and a CircuitBreaker.
  
除了失败率阈值 `failureRateThreshold`，还提供了类似慢调用耗时阈值 `slowCallRateThreshold` ，触发统计为 `Open` 的最小失败调用次数阈值
`minimumNumberOfCalls` 以及可以忽略的异常类型 `ignoreException` 等， 更多参数可以参考[doc](https://resilience4j.readme.io/docs/circuitbreaker)。

### References
- https://resilience4j.readme.io
- https://medium.com/@storozhuk.b.m/circuit-breaker-implementation-in-resilience4j-992af908c413
- https://medium.com/@storozhuk.b.m/achieving-fault-tolerance-with-resilience4j-21fcd7fef6c
- https://www.liangzl.com/get-article-detail-10088.html
- https://ucare.cs.uchicago.edu/pdf/socc14-cbs.pdf



> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
