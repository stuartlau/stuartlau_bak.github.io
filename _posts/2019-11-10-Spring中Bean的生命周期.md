---
layout:     post
title:      "谈Spring中Bean的生命周期管理"
subtitle:   "Bean Management in Spring"
date:       2019-11-10
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Spring
---
> Spring的IoC容器功能非常强大，负责Spring的Bean的创建和管理等功能。了解Spring Bean的生命周期对我们了解整个Spring框架会有很大的帮助。
>
> 本文主要介绍ApplicationContext和BeanFactory两种容器的Bean的生命周期。 


### Bean的生命周期
Bean 的生命周期由多个特定的生命阶段组成，每个生命阶段都开除了一扇门，允许外接对 Bean 施加控制。

在 Spring 中，我们可以从两个层面定义 Bean 的生命周期：第一个层面是 Bean 的作用范围；第二个层面是是实例化 Bean 时所经历的一系列阶段。

BeanFactory和ApplicationContext是Spring两种很重要的容器，前者提供了最基本的依赖注入的支持，而后者在继承前者的基础进行了功能的拓展，例如增加了事件传播，资源访问和国际化的消息访问等功能。
#### BeanFactory中Bean的生命周期
1. 当调用者通过 getBean(beanName) 向容器请求某一个 Bean 时，如果容器注册了 `InstantiationAwareBeanPostProcessor` 接口，
在实例化 Bean 之前，将调用接口的 postProcessBeforeInstantiation()方法；
2. 根据配置情况调用 Bean 构造函数或工厂方法实例化 Bean；
3. 如果容器注册了 `InstantiationAwareBeanPostProcessor` 接口，在实例化 Bean 之后，调用该接口的 
postProcessAfterInstantiation()方法，在这里可以对已经实例化的对象进行一些「梳妆打扮」；
4. 如果 Bean 配资了属性信息，容器在这一步着手将配置值设置到 Bean 对应的属性中，不过在设置每个属性之前将先调用 
`InstantiationAwareBeanPostProcessor` 接口的
postProcessPropertyValues()方法； 
5. 调用 Bean 的属性设置方法设置属性值；
6. 如果 Bean 实现了 `BeanNameAware` 接口，将调用 setBeanName()方法，将配置文件中该 Bean 对应的名称设置到 Bean 中；
7. 如果 Bean 实现了 `BeanFactoryAware` 接口，将调用 setBeanFactory()接口方法，将 BeanFactory 容器设置到 Bean 中；
8. 如果 BeanFactory 装配了 `BeanPostProcessor` 后处理器，将调用 postProcessBeforeInitialization(Object bean, 
String beanName)接口方法对 Bean 进行加工操作，一些常见的功能如AOP、动态代理等都是通过这个接口实现的；
9. 如果 Bean 实现了 `InitializingBean` 接口，将调用接口的 afterPropertiesSet()方法；
10. 如果在<bean>标签里设置了 init-method 属性则执行这个方法；
11. 如果 BeanFactory 装配了 `BeanPostProcessor` 后处理器，将调用 postProcessAfterInitialization(Object 
bean, String beanName)接口方法对 Bean 进行加工操作；
12. 如果设置scope=prototype，将 Bean 返回给调用者，不再负责其生命周期，如果是singleton类型，则放入 Spring IoC 容器的缓存池中，并将引用返回给调用者，
Spring 继续对这些 Bean 进行后续的生命管理；
13. 对于设置scope=singleton，当容器关闭时，将触发 Spring 对 Bean 的后续生命周期的管理工作，如果实现了 `DisposableBean` 
接口，则调用destroy()方法，在此处可以编写如资源释放、日志记录等工作；
14. 如果在<bean>标签中配置了 destroy-method 属性，则将执行这个方法，完成 Bean 资源的释放；

下面对这些关键点的特定方法进行一定的分类：
- Bean 自身的方法：如构造函数实例化 Bean，调用 Setter设置属性，以及通过 <bean> 标签设置init-method和destroy-method所指定的方法；
- Bean 级声明周期接口方法：如 BeanNameAware、BeanFactoryAware、InitializingBean、DisposableBean，这些接口由 Bean 类直接实现；
- 容器级生命周期接口方法：包括 InstantiationAwareBeanPostProcessor和BeanPostProcessor 两个接口，这些称之为后处理器，一般不由 Bean 
本身实现，它们独立于 Bean，实现类以容器附加装置的形式注册到 Spring 容器中并通过接口反射为 Spring 容器预先识别。这些后处理器在任何 Bean 创建的时候
都会发生作用，影响是全局性的。

> 如果喜欢解绑Spring，可以使用init-method替换实现InitializingBean接口，或者使用destroy-method替换实现DisposableBean接口。
>
> 甚至还可以使用@PostConstruct、@PreDestroy注解来完成相同的方法标记工作，它们是由InitDestroyAnnotationBeanPostProcessor后处理器实现的。

```java
package org.springframework.beans.factory;

public interface BeanFactory {

    /**
     * 用来引用一个实例，或把它和工厂产生的Bean区分开，就是说，如果一个FactoryBean的名字为a，那么，&a会得到那个Factory
     */
    String FACTORY_BEAN_PREFIX = "&";

    /*
     * 四个不同形式的getBean方法，获取实例
     */
    Object getBean(String name) throws BeansException;

    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    <T> T getBean(Class<T> requiredType) throws BeansException;

    Object getBean(String name, Object... args) throws BeansException;

    boolean containsBean(String name); // 是否存在

    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;// 是否为单实例

    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;// 是否为原型（多实例）

    boolean isTypeMatch(String name, Class<?> targetType)
            throws NoSuchBeanDefinitionException;// 名称、类型是否匹配

    Class<?> getType(String name) throws NoSuchBeanDefinitionException; // 获取类型

    String[] getAliases(String name);// 根据实例的名字获取实例的别名

}

```
BeanFactory有着庞大的继承、实现体系，有众多的子接口、实现类。来看一下BeanFactory的基本类体系结构（接口为主）：
![继承体系](https://images2015.cnblogs.com/blog/249993/201609/249993-20160907110538348-921805562.png)
#### ApplicationContext中Bean的生命周期

Bean 在ApplicationContext 中的生命周期和在 BeanFactory 中生命周期类似，不同的是，如果 Bean 实现了 ApplicationContextAware 
接口，会增加一个调用该接口的setApplicationContext()方法的步骤。
此外，如果配置文件中声明了工厂后处理器接口 BeanFactoryPostProcessor 的实现类，则应用上下文在装载配置文件之后初始化 Bean 实例之前将调用
这些 BeanFactoryPostProcessor 对配置信息进行加工处理，如 Spring 自带的后工厂处理器 
PropertyPlaceholderConfigurer、CustomEditorConfigurer等。

ApplicationContext 和 BeanFactory 另一个最大的不同之处在于：前者会利用 Java 反射机制自动识别出配置文件中定义的 
BeanPostProcessor、InstantiationAwareBeanPostProcessor和 BeanFactoryPostProcessor，并自动将它们注册到应用上下文中；
后者需要在代码中通过手动调用 addBeanPostProcessor()方法进行注册。

所以一般我们都是用 ApplicationContext，只需要在配置文件中配置对应的 <bean> 即可。

ApplicationContext接口作为BeanFactory的派生，提供BeanFactory所有的功能。而且ApplicationContext
还在功能上做了扩展，相较于BeanFactory，ApplicationContext还提供了以下的功能： 

-（1）MessageSource, 提供国际化的消息访问  
-（2）资源访问，如URL和文件  
-（3）事件传播特性，即支持aop特性
-（4）载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的Web层 

看一下接口声明就知道了：
```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {...}
```
### BeanFactory对比ApplicationContext
#### 特性对比
BeanFactory主要是面对与 Spring 框架的基础设施，面对 Spring 自己。而 ApplicationContext 主要面对与 Spring 使用的开发者。基本都会使用 
ApplicationContext 并非 BeanFactory。

通过BeanFactory启动IoC容器时，并不会初始化配置文件中定义的Bean，初始化动作发生在第一个调用时。

BeanFactory接口提供了配置框架及基本功能，但是无法支持 Spring 的 AOP 功能和 Web 应用。

BeanFactory采用的是「延迟加载」形式来注入Bean的，即只有在使用到某个Bean时(调用getBean()方法)
，才对该Bean进行加载实例化，这样我们就不能及时发现一些存在的Spring的配置问题。而ApplicationContext则相反，它是在容器启动时，一次性创建了所有的Bean
。这样，在容器启动时，我们就可以发现Spring中存在的配置错误。 相对于基本的BeanFactory，ApplicationContext 唯一的不足是占用内存空间。当应用程序配置Bean较多时，程序启动较慢。

BeanFactory和ApplicationContext都支持BeanPostProcessor、BeanFactoryPostProcessor
的使用，但两者之间的区别是：BeanFactory需要手动注册，而ApplicationContext则是自动注册。

#### 使用方式对比    
   
- BeanFactory
```java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
BeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
beanDefinitionReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));
DemoServiceImpl demo = (DemoServiceImpl) beanFactory.getBean("demo");
```    
- ApplicationContext
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
DemoService demo = (DemoService) ctx.getBean("demo");
```
可以看到使用 ApplicationContext 更简单。

> 两者比较看出，BeanFactory 其实只有基本只有跟 Bean 相关的功能。所以在实际开发中，我们使用 ApplicationContext 就可以使用相关 Spring 功能。如果使用
 BeanFactory 其实也可以也实现这些功能，但是这个时候我们就需要知道其中很多相关细节。作为一个框架而言，其自然做到是开箱即用，而无需让使用者考虑其内部相关细节。所以 Spring 
 做了相关封装，最后成了我们经常使用的 ApplicationContext。
 
### References
- 《Spring 3.x——企业应用开发实战》，陈雄华、林开雄著
- [Spring系列之beanFactory与ApplicationContext](https://www.cnblogs.com/xiaoxi/p/5846416.html)

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
