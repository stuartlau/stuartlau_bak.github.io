---
layout:     post
title:      "AOP的概念结合Spring的解析"
subtitle:   "AOP"
date:       2020-03-28
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - Spring
    - AOP
---
    
> AOP是编程中常用的一种方式，以Spring为例我们经常使用spring-aop组件和aspectJ的注解或者配置文件来完成对接口或类中的
某些方法的执行的拦截，本文对用到的一些概念术语进行解读，方便理解。

![](https://i.stack.imgur.com/J7Hrh.png)
### 概念
#### Joinpoint
> A joinpoint is a candidate point in the Program Execution of the application where an aspect can 
be plugged in. This point could be a method being called, an exception being thrown, or even a field 
being modified. These are the points where your aspect’s code can be inserted into the normal flow 
of your application to add new behavior.

注意 `Joinpoint` 并不一定只是方法的执行点，还可以是一个异常的抛出点或者一个属性的变更点，在这些变动上我们都可以进行拦截。

下面用一个餐馆的例子来比喻：
> Join points are the options on the menu and pointcuts are the items you select. A joinpoint is an 
opportunity within code for you to apply an aspect…just an opportunity. Once you take that 
opportunity and select one or more joinpoints and apply an aspect to them, you’ve got a pointcut.
  
`Joinpoint` 就是菜单上的选项，`Pointcut` 就是你选的菜。`Joinpoint` 只是你切面中可以切的那些方法，一旦你选择了要切哪些方法，
那就是 `Pointcut` 了。

也就是说，所有在你程序中可以被调用的方法都是 `JoinPoint` 。 使用 `Pointcut` 表达式，那些匹配了的方法，才叫 `Pointcut`。
所以你根本不用关心 `JoinPoint` 。

再通俗一点，比如你有10个方法，你只想切2个方法，那么那10个方法就是 `Joinpoint` ， 2个方法就是 `PointCut` 。

下面是spring-aop中对 `Jointpoint` 接口的定义：
```java
public interface Joinpoint {

	/**
	 * Proceed to the next interceptor in the chain.
	 * <p>The implementation and the semantics of this method depends
	 * on the actual joinpoint type (see the children interfaces).
	 * @return see the children interfaces' proceed definition
	 * @throws Throwable if the joinpoint throws an exception
	 */
	Object proceed() throws Throwable;

	/**
	 * Return the object that holds the current joinpoint's static part.
	 * <p>For instance, the target object for an invocation.
	 * @return the object (can be null if the accessible object is static)
	 */
	Object getThis();

	/**
	 * Return the static part of this joinpoint.
	 * <p>The static part is an accessible object on which a chain of
	 * interceptors are installed.
	 */
	AccessibleObject getStaticPart();

}

```
> This interface represents a generic runtime joinpoint (in the AOP terminology). A runtime joinpoint 
is an <i>event</i> that occurs on a static joinpoint (i.e. a location in a the program). For instance, 
an invocation is the runtime  joinpoint on a method (static joinpoint). The static part of a given 
joinpoint can be generically retrieved using the {@link #getStaticPart()} method.
>
> In the context of an interception framework, a runtime joinpoint is then the reification of an 
access to an accessible object (a method, a constructor, a field), i.e. the static part of the
joinpoint. It is passed to the interceptors that are installed on the static joinpoint.
 
注释里面有一个概念：static joinpoint，比如方法就是一种，可以通过 `getStaticPart` 方法获取一个 *joinpoint* 的
 *static part* 。

对于 `getThis` 方法来说需要注意的一点，如果它返回的是持有当前 *jointpoint* 的static part的对象，比如一个触发的目标对象，
如果是静态方法则返回null。

这里提到了 `Invocation` ，看一下它的接口定义：
```java
public interface Invocation extends Joinpoint {

	/**
	 * Get the arguments as an array object.
	 * It is possible to change element values within this
	 * array to change the arguments.
	 * @return the argument of the invocation
	 */
	Object[] getArguments();

}
```
> This interface represents an invocation in the program. An invocation is a joinpoint and can be 
intercepted by an interceptor.

`Invocation` 即「调用」，它也是一种 *jointpoint* ，主要用于 `Interceptor` 拦截器拦截的时候使用。

它的接口 `getArgument` 会返回当前被执行的方法的入参，以数组的形式返回。

一种比较常见的子接口是 `MethodInvocation` ：
```java
public interface MethodInvocation extends Invocation {

	/**
	 * Get the method being called.
	 * <p>This method is a friendly implementation of the
	 * {@link Joinpoint#getStaticPart()} method (same result).
	 * @return the method being called
	 */
	Method getMethod();

}
```
> Description of an invocation to a method, given to an interceptor upon method-call.
A method invocation is a joinpoint and can be intercepted by a method interceptor.

可以看到一个 *method invocation* 就是一个 *jointpoint*，即方法调用类型的 *joinpoint* ，它会拦截器拦截，同时它的
方法 `getMethod` 返回的和 `Jointpoint` 的方法 `getStaticPart` 的返回结果是一样的，都是当前被执行的方法。

关于拦截器 `Interceptor` 下面讲 `Advice` 的时候会再详细介绍。
#### Pointcut
> A pointcut defines at what joinpoints, the associated Advice should be applied. 
Advice can be applied at any joinpoint supported by the AOP framework. Of course, 
you don’t want to apply all of your aspects at all of the possible joinpoints. 
Pointcuts allow you to specify where you want your advice to be applied. 
Often you specify these pointcuts using explicit class and method names or 
through regular expressions that define matching class and method name patterns. 
Some AOP frameworks allow you to create dynamic pointcuts that determine whether 
to apply advice based on runtime decisions, such as the value of method parameters.

区别如下：
> Joinpoint - Potential places to apply/run the advice code.
> 
> Pointcut - actual chosen joinpoints for executing the advice.

spring-aop中的 `Pointcut` 接口定义如下：
```java
public interface Pointcut {

	/**
	 * Return the ClassFilter for this pointcut.
	 * @return the ClassFilter (never {@code null})
	 */
	ClassFilter getClassFilter();

	/**
	 * Return the MethodMatcher for this pointcut.
	 * @return the MethodMatcher (never {@code null})
	 */
	MethodMatcher getMethodMatcher();


	/**
	 * Canonical Pointcut instance that always matches.
	 */
	Pointcut TRUE = TruePointcut.INSTANCE;

}
```
它需要提供 `MethodMatcher` 实现类和 `ClassFilter` 实现类分别用于匹配方法和类，二者要同时满足才会被增强，如我想拦截
所有Grpc服务类的非 *Object* 方法，或者带有某个类注解的所有方法等。
#### Advice
> This is an object which includes API invocations to the system wide concerns 
representing the action to perform at a joinpoint specified by a point.

中文翻译为「声明」(也有叫「通知」的，但注解里面也说到了是一个*action to perform*，所以个人认为叫增强更好)， 
*advice* 就是你作用到 *pointcut* 上的方式和行为，如可以使用Before, After或者Around等方式，以及对应的相应的代码逻辑。

spring-aop中的定义如下：
```java
/**
 * Tag interface for Advice. Implementations can be any type
 * of advice, such as Interceptors.
 */
public interface Advice {

}
```
是的，一个空的接口定义，只是一个 `Tag interface` ，实现类可以是任何的 *advice* ，比如拦截器 `Interceptor` 也是一种。

基于该接口衍生出了很多子接口如： `MethodBeforeAdvice` 、 `AfterReturningAdvice` 、 `ThrowsAdvice` 等。

#### Interceptor
我们来看一下一种常见的 `Advice` 拦截器 `Interceptor` 的定义：
```java
public interface Interceptor extends Advice {

}
```
这是一个通用的接口，一般不会直接使用而是使用它的一些子接口（如MethodInterceptor），文档中对它的解释重点如下：
> A generic interceptor can intercept runtime events that occur within a base program. Those 
events are materialized by (reified in) joinpoints. Runtime joinpoints can be invocations, field
 access, exceptions
 
可以看到它是用来拦截运行时程序中的事件event的，这些事件以 *jointpoint* 的「物化形式」存在，比如调用（Invocations）、
属性访问（Field Access）和异常（Exceptions）。

看一下我们常用的方法拦截器 `MethodInterceptor` 接口的定义：
```java
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {

	/**
	 * Implement this method to perform extra treatments before and
	 * after the invocation. Polite implementations would certainly
	 * like to invoke {@link Joinpoint#proceed()}.
	 * @param invocation the method invocation joinpoint
	 * @return the result of the call to {@link Joinpoint#proceed()};
	 * might be intercepted by the interceptor
	 * @throws Throwable if the interceptors or the target object
	 * throws an exception
	 */
	Object invoke(MethodInvocation invocation) throws Throwable;

}
```
可以看到它本质上是抽象了对拦截到的「方法调用」的处理接口，入参是 `MethodInvocation` ，一般在执行 `Jointpoint#proceed()` 
之前和之后进行处理，返回结果可以是方法的返回值或hack后的返回值。
#### Advisor
*advisor* 就是作用在具体对象上的 *pointcut* 和 *advice* ，把 *pointcut* 和 *advice* 合起来就是 *advisor* 。
比如切到哪个具体方法上面，是使用的Before、After还是Around，这个就叫 *advisor* 。

看一下spring-aop中的 `Advisor` 接口的定义：
```java
public interface Advisor {

	Advice EMPTY_ADVICE = new Advice() {};


	/**
	 * Return the advice part of this aspect. An advice may be an
	 * interceptor, a before advice, a throws advice, etc.
	 * @return the advice that should apply if the pointcut matches
	 */
	Advice getAdvice();

	boolean isPerInstance();

}
```
> Base interface holding AOP <b>advice</b> (action to take at a joinpoint) and a filter determining 
the applicability of the advice (such as a pointcut). <i>This interface is not for use by Spring 
users, but to allow for commonality in support for different types of advice.
>
> Spring AOP is based around <b>around advice</b> delivered via method <b>interception</b>, 
compliant with the AOP Alliance interception API.

乍一看Advice和Advisor接口没有什么大的区别，后者除了包括前者的一个实现类接口外还有一个是否是每个目标对象都会创建一个代理
的方法。

其实不是一个东西，看一下更通用的接口 `PointcutAdvisor` 就知道了：
```java
public interface PointcutAdvisor extends Advisor {

	/**
	 * Get the Pointcut that drives this advisor.
	 */
	Pointcut getPointcut();

}
```
> Superinterface for all Advisors that are driven by a pointcut. This covers nearly all advisors 
except introduction advisors, for which method-level matching doesn't apply.

这个接口才是我们要用到的那个接口，它提供一个接口要返回一个 `Pointcut` 对象，这个接口是通过 *pointcut* 来驱动 
的。比如，我们可以通过实现 `getPointcut` 方法返回一个只 `PointCut` 的实现类，让它的 `ClassFilter` 
和 `MethodMatcher` 实现的功能是： 对Grpc服务类的自有接口进行拦截的。 然后通过实现 `getAdvice` 方法返回一个 
`MethodInteceptor` 的实现类，默认对执行的方法通过Around的方式进行增强。

所以，Spring AOP中的 *advisor* 都是基于 *pointcut* 来驱动，并需要 *advice* 来完成具体的逻辑的。

#### Aspect
这个概念和Advisor其实差不多，它更多的出现在说明的文档中，以「切面」的名字示众。为什么说它也是Advisor呢，看一下Spring的
基于配置文件的注释就知道了：
```xml
<bean id="sysAspect" class="com.example.aop.SysAspect"/>
<!-- 配置AOP -->
<aop:config>
    <!-- 配置切点表达式  -->
    <aop:pointcut id="pointcut" expression="execution(public * com.example.controller.*Controller.*(..))"/>
    <!-- 配置切面及配置 -->
    <aop:aspect order="3" ref="sysAspect">
        <!-- 前置通知 -->
        <aop:before method="beforMethod"  pointcut-ref="pointcut" />
        <!-- 后置通知 -->
        <aop:after method="afterMethod"  pointcut-ref="pointcut"/>
        <!-- 返回通知 -->
        <aop:after-returning method="afterReturnMethod" pointcut-ref="pointcut" returning="result"/>
        <!-- 异常通知 -->
        <aop:after-throwing method="afterThrowingMethod" pointcut-ref="pointcut" throwing="ex"/>
        <aop:around method="aroundMethod" pointcut-ref="pointcut"/>
    </aop:aspect>
</aop:config>
```
该配置通过 `<aop:config>` 、 `<aop:pointcut>` 、 `<aop:aspect>`以及它的子标签 `<aop:before>` 、 `<aop:after>` 
等将一个类中的各个方法定义为具体的增强的逻辑（当然也可以直接在对应的类中使用各种注解来实现，这里只是用配置的方式做一个讲解）。
#### Target
目标对象就是织入 *advice* 的对象，也叫做 *advised object* ，由于Spring是通过运行时代理的方式来实现 *aspect* 的，
所以 *advised object* 总是一个代理对象（ *proxied object* ）。
#### Proxy
一个类被AOP织入 *advice* ，就会产生一个结果类，它是融合了原类和增强逻辑的代理类。在Spring AOP中，一个AOP代理类是一个
JDK动态代理对象或者一个CGLIB代理对象。

Spring为什么建议基于接口编程？因为它默认使用JDK的动态代理来完成AOP功能，如果是一个普通的类的方法，则只能使用CGLIB来实现。
但需要引入额外的asm的包。 

如果需要强制使用CGLIB，需要显示通过配置文件的方式设置 `<aop:config>` 标签对应的 `proxy-target-class` 为true：
```xml
<aop:config proxy-target-class="true">
    <!-- other beans defined here... -->
</aop:config>
```
如果是使用@AspectJ注解则需要设置 `<aop:aspectj-autoproxy>` 标签对应的属性为true：
```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```
### Spring AOP和AspectJ
Spring AOP的目的是提供一个能个Spring IOC紧密集成的代理方式，解决常见的企业及开发应用中的问题，并不像AspectJ那样强大，
比如细粒度的对象的增强Spring AOP就做不了。

> Spring seamlessly integrates Spring AOP and IoC with AspectJ, to enable all uses of AOP within a 
consistent Spring-based application architecture. This integration does not affect the Spring AOP 
API or the AOP Alliance API. Spring AOP remains backward-compatible.

Spring AOP用起来比较简单，虽然它也需要依赖aspectweaver.jar，但是也只是借用了其中的注解和语法。它不需要引入AspectJ
的compiler/weaver到开发和构建的环节中。如果仅需要对Spring Bean进行切面，直接用Spring AOP即可，如果不是Spring容器
管理的对象，比如领域对象，则可以考虑使用AspectJ，对于简单方法执行的切面Spring AOP可以搞定，但是对于诸如属性获取和设置
则需要使用AspectJ来完成。
#### @AspectJ支持
Spring有两种方式使用@AspectJ注解来实现AOP，该注解是在AspectJ 5中引入的，需要保证aspectjweaver.jar在classpath中。

使用Java Configuration的方式：
```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig{
}
```

使用XML方式：
```xml
<aop:aspectj-autoproxy/>
```

> 需要注意的是@AspectJ注解本身并不会被Spring包扫描自动发现，需要使用如@Component注解来使Spring发现。

#### 例子
除了通过配置文件声明aspects，如使用 `<aop:config>` 或者 `<aop:aspectj-autoproxy>` 之外，还可以通过编程的方式
创建proxies来完成对目标对象的advised。

一般用AspectJ风格的pointcut表达式我们经常使用，如@Aspect、@Before等这里不再深入，如果用编程的方式实现可以利用 
`ProxyFactory` 类来实现编程方式的AOP功能。比如下面的例子：
```java
import org.springframework.aop.MethodBeforeAdvice;
import java.lang.reflect.Method;

public class MethodBeforeAdviceBarImpl implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("Bar!");
    }
}
```

使用Spring AOP自带的Advisor实例，只需要引入spring-context.jar即可：
```java
import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.NameMatchMethodPointcutAdvisor;

public class App {

    public static void main(String[] args) {
        final MethodBeforeAdvice advice = new MethodBeforeAdviceBarImpl();

        final NameMatchMethodPointcutAdvisor nameMatchMethodPointcutAdvisor = new NameMatchMethodPointcutAdvisor();
        nameMatchMethodPointcutAdvisor.setMappedName("foo");
        nameMatchMethodPointcutAdvisor.setAdvice(advice);

        final ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.addAdvisor(nameMatchMethodPointcutAdvisor);

        final Foo foo = new FooImpl();
        proxyFactory.setTarget(foo);

        final Foo fooProxy = (Foo) proxyFactory.getProxy();
        fooProxy.foo();
    }
}

```

但是如果要使用AspectJ表达式形式的Advisor还需要引入aspectjweaver.jar（否则在运行时会报错），比如下面的例子：
```java
import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor;
import org.springframework.aop.framework.ProxyFactory;

public class App {

    public static void main(String[] args) {
        final MethodBeforeAdvice advice = new MethodBeforeAdviceBarImpl();

        final AspectJExpressionPointcutAdvisor aspectJExpressionPointcutAdvisor = new AspectJExpressionPointcutAdvisor();
        aspectJExpressionPointcutAdvisor.setAdvice(advice);
        aspectJExpressionPointcutAdvisor.setExpression("execution(void biz.tugay.spashe.Foo.foo())");

        final ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.addAdvisor(aspectJExpressionPointcutAdvisor);

        final Foo foo = new FooImpl();
        proxyFactory.setTarget(foo);

        final Foo fooProxy = (Foo) proxyFactory.getProxy();
        fooProxy.foo();
    }
}
```
`AspectJExpressionPointcutAdvisor` 内部通过 `AspectJExpressionPointcut` 来实现，它引入了很多AspectJ的包：
```java
import org.aopalliance.intercept.MethodInvocation;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.aspectj.weaver.patterns.NamePattern;
import org.aspectj.weaver.reflect.ReflectionWorld.ReflectionWorldException;
import org.aspectj.weaver.reflect.ShadowMatchImpl;
import org.aspectj.weaver.tools.ContextBasedMatcher;
import org.aspectj.weaver.tools.FuzzyBoolean;
import org.aspectj.weaver.tools.JoinPointMatch;
import org.aspectj.weaver.tools.MatchingContext;
import org.aspectj.weaver.tools.PointcutDesignatorHandler;
import org.aspectj.weaver.tools.PointcutExpression;
import org.aspectj.weaver.tools.PointcutParameter;
import org.aspectj.weaver.tools.PointcutParser;
import org.aspectj.weaver.tools.PointcutPrimitive;
import org.aspectj.weaver.tools.ShadowMatch;
```

#### exposeProxy属性
在 `ProxyConfig` 类中有一个 *exposeProxy* 属性， 默认为false，它的含义如下：

> Set whether the proxy should be exposed by the AOP framework as a ThreadLocal for retrieval via 
the AopContext class. This is useful if an advised object needs to call another advised method on 
itself. (If it uses {@code this}, the invocation will not be advised). Default is "false", in 
order to avoid unnecessary extra interception. This means that no guarantees are provided that 
AopContext access will work consistently within any method of the advised object.

如果不设置为true，想通过 `AopContext` 类来获取当前代理对象时会报错：
> java.lang.IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available.

设置的方法根据版本不同而不同
- Spring配置文件

```xml
<aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true"/>
```

- SpringBoot

```java
@EnableAspectJAutoProxy(exposeProxy=true, proxyTargetClass=true)
```

这引出了一个很经典的问题，即用Spring AOP进行advised过的代理对象在调用目标对象的方法时是可以被增强的，但其内部如果再调用本身对象的方法，则该方法
是不会被增强的。

`AopContext` 的使用方式如下（来自官网），它可以获得当前线程的上下文中的代理对象：

```java
public class SimplePojo implements Pojo {

    public void foo() {
        // this works, but... gah!
        ((Pojo) AopContext.currentProxy()).bar();
    }

    public void bar() {
        // some logic...
    }
}
```
但是这种方式会强制耦合Spring AOP，入侵性强不说，还让上下文显示的知道当前代码要被用在AOP的上下文中，并且需要设置 
*exposeProxy* 属性为true，否则 `AopContext` 拿不到当前执行的代理对象，也就无法触发植入的逻辑。

Spring AOP通过如下代码将代理对象放入到`Aop`具体代码参考：
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    ... ...

    Object retVal;

    if (this.advised.exposeProxy) {
        // Make invocation available if necessary.
        oldProxy = AopContext.setCurrentProxy(proxy); // 内部是一个ThreadLocal对象来维护
        setProxyContext = true;
    }

    ... ...
```

除了使用上面的方法外，还有一种比较简洁但是很奇葩的做法就是在当前的Bean中注入自己，然后在相关方法调用自身方法的地方使用
这个注入的实例来调用相应的方法，即不使用this（隐式），因为this其实是被代理对象本身，需要使用proxy对象进行调用实现增强，
如果拿不到当前对象的代理，就自己注入一个，其实和上面的 `AopContext.currentProxy()` 是一个原理。

> 需要注意AspectJ就没有这样的问题，因为它并不是一个基于代理的AOP框架，而是编译和加载时就已经植入了代码到源码中。

其他比较经典的关于AOP失效的例子包括 `@Async` 失效、 `@Transactional` 失效等，都是通过对当前类对象来调用具体的被声明为增强的方法
但是无法成功。

#### 多线程和事务管理
Spring的事务处理为了与数据访问解耦，它提供了一套处理数据资源的机制，而这个机制与上文中的原理相差无几，也是采用的ThreadLocal的方式。

在编程中，Service实例都是单例的无状态的，事务管理则需要加入事务控制的相关状态变量，使得Service实例不再是无状态线程安全的，解决这个问题的方式就是使用ThreadLocal。

通过使用ThreadLocal将数据源绑定在当前线程上，在当前线程的事务中，从设定的地方去取连接就会是同一个数据库连接，这样操作事务就会在同一个连接上进行。

但是，ThreadLocal的特性是，绑定在当前线程中的变量不会自动传递到其它线程中（当然，InheritableThreadLocal可以在父子线程中间传递变量值，但是这需要特殊的使用场景），所以当开启子线程时，子线程并没有父线程的数据库连接资源。

### References
- https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-introduction-defn
- https://stackoverflow.com/questions/15447397/spring-aop-whats-the-difference-between-joinpoint-and-pointcut
- https://stackoverflow.com/questions/11446893/spring-aop-why-do-i-need-aspectjweaver
- https://www.cnblogs.com/duanxz/p/4367362.html
- https://stackoverflow.com/questions/6222600/transactional-method-calling-another-method-without-transactional-anotation

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
