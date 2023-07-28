---
layout:     post
title:      "把Future扔进CompletableFuture的封装里"
subtitle:   "CompletableFuture in Java"
date:       2019-11-10
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - CompletableFuture
---
> 关于异步编程，之前一直在项目中使用Guava的ListenableFuture，对于JDK8的CompletableFuture使用较少。
>
> 注：这篇文章绝大部分内容都来自colobu.com中的《Java CompletableFuture 详解》的介绍，感谢作者。后续我会统一改成自己的内容。

### Future In JDK5
Future是JDK5添加的类，用来描述一个异步计算的结果。可以用isDone方法来检查计算是否完成，或者使用get阻塞住调用线程，直至计算完成返回结果，也可以用cancel方法来停止任务的执行。

```java
public class BasicFuture {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService es = Executors.newFixedThreadPool(10);
        Future<Integer> f = es.submit(() -> {
            // 长时间的异步计算
            // ...
            // 然后返回结果
            return 100;
        });
        f.get();
    }
}
```
Future以及相关使用方法提供了异步执行任务的能力，但对于结果的获取确实不方便，只能通过阻塞或轮询的方式得到任务结果。
阻塞的方式与我们理解的异步编程其实是相违背的，而轮询又会耗CPU
资源，而且还不能及时得到计算结果，为什么不能用「观察者设计模式」当计算结果完成即时通知监听者呢？

### Future in Netty
很多语言像Node.js，采用回调的方式实现异步编程。Java的一些框架像Netty，自己扩展Java的Future接口，提供了addListener等多个扩展方法：

```java
ChannelFuture future = bootstrap.connect(new InetSocketAddress(host, port));
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception{
      if (future.isSuccess()) {
          // SUCCESS
      }
      else {
          // FAILURE
      }
    }
});
```
### CompletableFuture in JDK8
在Java 8中, 新增加了一个包含50个方法左右的类——CompletableFuture，提供了非常强大的Future
的扩展功能，可以帮助我们简化异步编程的复杂性，提供了「函数式编程」的能力，可以通过「回调」的方式处理计算结果，并且提供了「转换」和「组合」CompletableFuture的方法。
#### CompletionStage
这个是最基础的接口，CompletionFuture是它的一个子接口，来看一下JavaDoc中的介绍：
``` 
A stage of a possibly asynchronous computation, that performs an action or computes a value when another 
CompletionStage completes. A stage completes upon termination of its computation, but this may in turn trigger other dependent stages. 
```
所以每一个执行过程都称为一个stage，它的执行以来前面的stage的完成，并可能影响后面的dependent stages，同时每个stage都可以是异步执行的。

每一个stage的计算都可以以来下面的任何一种方式：Function、Consumer或Runnable，对应的方法依次以apply、accept或run
开头，这也方便我们记忆，主要区别是stage是否需要参数以及是否需要生成新的结果，如：
> stage.thenApply(x -> square(x)).thenAccept(x -> System.out.print(x)).thenRun(() -> System.out.println())

#### 主动完成计算
CompletableFuture类实现了CompletionStage和Future接口，所以你还是可以像以前一样通过阻塞或者轮询的方式获得结果，
尽管这种方式不推荐使用。

```java
public T get() throws InterruptedException, ExecutionException
public T get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException
public T getNow(T valueIfAbsent)
public T join()
```
getNow有点特殊，如果结果已经计算完则返回结果或者抛出异常，否则返回给定的valueIfAbsent值。

join返回计算的结果或者抛出一个unchecked异常(CompletionException，它包含对应的原始异常)，它和get对抛出的异常的处理有些细微的区别，
你可以运行下面的代码进行比较：
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    int i = 1/0;
    return 100;
});
//future.join();
future.get();
```
区别主要在方法声明上，一个是声明了检查异常，一个没有声明会抛出非检查异常。并且抛出的异常的类型也是不同的，
一个是ExecutionException，一个是CompletionException，二者均对原始异常
进行了包装。
#### 创建对象
CompletableFuture.completedFuture是一个静态辅助方法，用来返回一个已经计算好的CompletableFuture：
```java
public static <U> CompletableFuture<U> completedFuture(U value)
```
注意，这个返回的结果已经是计算好的，直接获取可以得到结果。

而以下四个静态方法用来执行一段「异步执行」的代码（通过Runnable或者Supplier）来创建CompletableFuture对象：
```java
public static CompletableFuture<Void>   runAsync(Runnable runnable)
public static CompletableFuture<Void>   runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U>  supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U>  supplyAsync(Supplier<U> supplier, Executor executor)
```
以Async结尾并且没有指定Executor的方法会使用 `ForkJoinPool.commonPool()` 作为它的线程池执行异步代码。

runAsync方法也好理解，它以Runnable函数式接口类型为参数，所以CompletableFuture的计算结果为空类型Void。

supplyAsync方法以Supplier<U>函数式接口类型为参数，CompletableFuture的计算结果类型为Supplier的返回类型U。

#### whenComplete
当CompletableFuture的计算正常完成，或者计算时抛出异常时，我们可以执行whenComplete方法来执行特定的Action。主要是下面的方法：
```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
```
可以看到Action的类型是 `BiConsumer<? super T,? super Throwable>` 
，即回调该方法时传入的包括返回结果和可能的异常对象。

方法不以Async结尾，意味着Action使用触发当前计算CompletableFuture执行的相同的线程执行。

> 注意这几个方法都会返回CompletableFuture，当Action执行完毕后它的结果返回原始的CompletableFuture的计算结果或者返回异常，
需要注意的是不要在该方法中对返回的结果进行「变更」，虽然该方法并不会影响最终返回的CompletableFuture对象实例，
但是可以改变它持有的结果对象的value内容（引用），如果想变更返回的value的内容，如改变类型，可以使用下面介绍的thenApply方法。

这里单独提一下有个exceptionally方法，可以通过chain的方式调用它，但它只有在CompletableFuture计算出现异常时才会被执行：
```java
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```
[例子](https://stackoverflow.com/questions/31338514/functional-java-interaction-between-whencomplete-and-exceptionally)：
```java
CompletableFuture<String> test=new CompletableFuture<>();
test.whenComplete((result, ex) -> System.out.println("stage 2: "+result+"\t"+ex))
    .exceptionally(ex -> { System.out.println("stage 3: "+ex); return ""; });
test.completeExceptionally(new IOException()); // 这个方法设置stage状态为完成并设置返回的异常
```
输出结果：
```
stage 2: null   java.io.IOException
stage 3: java.util.concurrent.CompletionException: java.io.IOException
```
因为采用了链式处理，所以先处理whenComplete后处理exceptionally，并且exceptionally的触发是由于显式调用了completeExceptionally
方法让计算抛出异常才得以被触发，但是whenComplete无论什么结果都会被触发。

如果不用链式的方式，可能在并发环境下由于执行的顺序会不一样导致结果不一样，具体可以参考上面的例子链接。
#### handle
它和whenComplete的区别主要是它可以有返回值并且可以改变返回的数据类型，它的参数是BiFunction而不是BiConsumer：
```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```
所以，它本质上可以对返回值和异常进行处理并返回新结果，相当于增强版本的whenComplete+exceptionally，因为它还可以改变返回数据类型。
这个有点像我们下面介绍的thenApply，只不过后者不能处理异常并且在前面的CompletableFuture计算中出现异常时不会触发后面的thenApply，而
handle并不受异常的影响。
#### thenApply
CompletableFuture可以作为monad(单子)和functor。由于回调风格的实现，我们不必因为等待一个计算完成而阻塞着调用线程，
而是告诉CompletableFuture当计算完成的时候请执行某个Function。而且我们还可以将这些操作串联起来，或者将CompletableFuture组合起来。

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```
这一组函数的功能是当原来的CompletableFuture计算完后，将结果传递给函数fn，将fn的结果作为新的CompletableFuture计算结果。
因此它的功能相当于将CompletableFuture<T>转换成CompletableFuture<U>并不改变future对象本身的引用。

> 注意Guava的Futures.transform(..)方法也提供了类似的功能，可以选择当前线程执行或使用独立的线程池执行。

使用例子如下：
```java
// 使用静态方法生成一个CompletableFuture对象
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
// 在future执行完毕后串联两个新的转换操作
CompletableFuture<String> f =  future.thenApplyAsync(i -> i * 10).thenApply(i -> i.toString());
System.out.println(f.get()); // "1000"
```
需要注意的是，这些转换操作并不是马上执行的，也不会阻塞（如果使用了Async），而是在前一个stage完成后继续执行。

下面来看一个链式调用多个操作的例子：
```java
CompletableFuture<String> futureA = CompletableFuture
                .supplyAsync(() -> "执行结果:" + (100 / 0))
                .thenApply(s -> "apply result:" + s)
                .whenComplete((s, e) -> {
                    if (s != null) {
                        System.out.println(s);//未执行
                    }
                    if (e == null) {
                        System.out.println(s);//未执行
                    } else {
                        System.out.println(e.getMessage());//java.lang.ArithmeticException: / by zero
                    }
                })
                .exceptionally(e -> {
                    System.out.println("ex"+e.getMessage()); //ex:java.lang.ArithmeticException: / by zero
　　　　　　　　　　　  return "futureA result: 100"; }); 
System.out.println(futureA.join());//futureA result: 100
```
运行可知，因为Supplier计算方法中出现了异常，导致后续的thenApply并不会被触发，而是直接运行到whenComplete
，即该方法不受异常的影响（毕竟它的参数就是支持可以处理异常的BiConsumer），最后因为发生了异常还会执行到exceptionally方法，
它可以设置一些默认返回结果等操作。

> 需要注意的是链式操作的顺序是可以随意改变的，exceptionally方法也可以放到whenComplete方法之前执行，
如果前者因为有异常被执行并返回了新的结果，则whenComplete方法将获取不到异常对象，只会得到前面的方法返回的结果，这点一定要注意！

#### thenAccept
上面介绍的方法是当计算完成的时候，会生成新的计算结果(如thenApply、 handle)
，或者返回同样的计算结果whenComplete，CompletableFuture还提供了一种处理结果的消费方法，只对结果执行Action而不返回新的计算值，
因此计算值为Void:

```java
public CompletableFuture<Void>  thenAccept(Consumer<? super T> action)
public CompletableFuture<Void>  thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void>  thenAcceptAsync(Consumer<? super T> action, Executor executor)
```
例子：
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<Void> f =  future.thenAccept(System.out::println);
System.out.println(f.get());
```

> 同样需要注意的是，这个方法在前面的CompletableFuture计算出异常时不会被触发，这一点显然不如用handle方便。

thenAcceptBoth以及相关方法提供了类似的功能，「当两个CompletionStage都正常完成计算的时候」，就会执行提供的
BiConsumer<? super T,? super U>类型的action，它用来将两个CompletionStage的结果进行统一处理。

runAfterBoth是当两个CompletionStage都正常完成计算的时候执行一个Runnable，因为run方法没有参数，所以它得不到计算的结果。
```java
public <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor)
public     CompletableFuture<Void> runAfterBoth(CompletionStage<?> other,  Runnable action)
```

例子如下：
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<Void> f =  future.thenAcceptBoth(CompletableFuture.completedFuture(10), (x, y) -> System.out.println(x * y));
System.out.println(f.get());  // null
```
更彻底地，下面一组方法当计算完成的时候会执行一个Runnable，与thenAccept不同，Runnable并不使用CompletableFuture计算的结果。

```java
public CompletableFuture<Void> thenRun(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
```
因此先前的CompletableFuture计算的结果被忽略了，这个方法返回CompletableFuture<Void>类型的对象。
> 这个方法在我们不关心CompletableFuture的结果的时候用比较方便，如只想打印完成时间。当然
也可以使用whenComplete或handle方法，但有点杀鸡用牛刀的感觉。


#### compose
```java
public <U> CompletableFuture<U> thenCompose(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn, Executor executor)
```
这一组方法接受一个Function作为参数，这个Function的输入是当前的CompletableFuture的计算结果值，
返回结果将是一个新的CompletableFuture，这个新的CompletableFuture会组合原来的CompletableFuture和函数返回的CompletableFuture。
因此它的功能类似：
> A +--> B +---> C

注意这里的A和B的执行顺序是固定的，A要等待B的执行完毕后才能组合成一个新的C。

例子：
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<String> f = future.thenCompose(i -> {
    return CompletableFuture.supplyAsync(() -> {
        return (i * 10) + "";
    });
});
System.out.println(f.get()); //1000

```
> 注意thenCompose和thenApply的区别，后者只是将CompletableFuture<T>转换成CompletableFuture<U>，
改变的是同一个CompletableFuture中的泛型类型。而thenCompose用来把当前CompletableFuture的结果作为参数连接到
另外一个CompletableFuture，并返回一个新的CompletableFuture。

#### combine
而下面的一组方法thenCombine用来复合另外一个CompletionStage的结果。它的功能类似：

A 
|
+------> B
+------> C
> 两个CompletionStage是并行执行的，它们之间并没有先后依赖顺序，*other*并不会等待先前的CompletableFuture执行完毕后再执行，
二者是独立执行的。

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
```
其实从功能上来讲它们的功能更类似thenAcceptBoth，只不过thenAcceptBoth是纯消费，它的函数参数没有返回值，而thenCombine的函数参数fn有返回值。
如果连两个任务的返回值也不关心，那么可以直接用runAfterBoth。
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "abc";
});
CompletableFuture<String> f =  future.thenCombine(future2, (x,y) -> y + "-" + x);
System.out.println(f.get()); //abc-100
```
注意，默认使用Async底层会用ForkJoinPool.commonPool()来执行，如果用thenCombine默认用当前线程执行，如果加了Async的方法会用ForkJoinPool
.commonPool()执行。
#### Either
thenAcceptBoth和runAfterBoth是当两个CompletableFuture都计算完成，而我们下面要了解的方法是当任意一个CompletableFuture计算完成的时候就会执行。
```java
public CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)
public <U> CompletableFuture<U>  applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn)
public <U> CompletableFuture<U>  applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn)
public <U> CompletableFuture<U>  applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor)
```
acceptEither方法是当任意一个CompletionStage完成的时候，action这个消费者就会被执行。这个方法返回CompletableFuture<Void>

applyToEither方法是当任意一个CompletionStage完成的时候，fn会被执行，它的返回值会当作新的CompletableFuture<U>的计算结果。
#### 异常处理
如果在设置 CompletableFuture.complete(value)之前出现了异常，那么 get() 或其他回调函数像 whenComplete() 都会无限期的等待下去。
```java
public static void main(string[] args) throws ExecutionException, InterruptedException {
    CompletableFuture<Double> futurePrice = new CompletableFuture<>();
    new Thread(() -> {
        if(true) {
            throw new RuntimeExeption("");
        }
        futurePrice.complete(23.5);
    }).start();
 
    System.out.println(futurePrice.get());
}
```
但是异常并不会在线程间传播，所以futurePrice.get()会一直等待。
解决方法1是调用get(timeout)方法，给定一个超时时间，如果指定时间内还没有获得结果则得到TimeoutException。
方法2是要在线程中通过completeExceptionally(ex)来传播异常：
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    CompletableFuture<Double> futurePrice = new CompletableFuture<>();
    new Thread(() -> {
        try {
            if (true) {
                throw new RuntimeException("Something wrong");
            }
            futurePrice.complete(23.5);
        } catch (Exception ex) {
            // 捕获的异常还会由ExecutionException包裹一下
            futurePrice.completeExceptionally(ex); 
        }
    }).start();
 
     System.out.println(futurePrice.get());
}
```
这时候在futurePrice.get()马上就能收到如下异常，注意get方法是返回ExecutionException异常而非CompletionException的：
> java.util.concurrent.ExecutionException: java.lang.Exception: Something wrong

这里有一个[例子](https://stackoverflow.com/questions/27430255/surprising-behavior-of-java-8-completablefuture-exceptionally-method)，
可以参考下。
#### 辅助方法
前面我们已经介绍了几个静态方法：completedFuture、runAsync、supplyAsync，下面介绍的这两个方法用来组合多个CompletableFuture。
```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```
- allOf方法是当所有的CompletableFuture都执行完后执行计算。
- anyOf方法是当任意一个CompletableFuture执行完后就会执行计算，计算的结果相同。

但是anyOf和applyToEither不同：
anyOf接受任意多的CompletableFuture但是applyToEither
只是判断两个CompletableFuture，anyOf返回值的计算结果是参数中其中一个CompletableFuture的计算结果，
applyToEither返回值的计算结果却是要经过fn处理的。



### References
- [Java CompletableFuture 详解](https://colobu.com/2016/02/29/Java-CompletableFuture/) 
- [Java 8: Definitive guide to CompletableFuture](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwivnI-BwJ7LAhWpg4MKHRr8CB0QFggcMAA&url=http%3A%2F%2Fwww.nurkiewicz.com%2F2013%2F05%2Fjava-8-definitive-guide-to.html&usg=AFQjCNHxOcm4uRrqZGl1ognxfaTtmB5k5A&sig2=A5rXKfQuabGJMHXAhPUIgA&bvm=bv.115339255,d.eWE)
- [JDK CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [JDK CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
