---
layout:     post
title:      "SpringMVC项目迁移SpringBoot的问题梳理"
subtitle:   "Migration SpringMVC Project to SpringBoot"
date:       2020-02-21
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Troubleshooting
    - SpringMVC
    - SpringBoot
---
    
> 本文主要梳理在将SpringMVC项目迁到SpringBoot框架下时遇到的各种编译、部署和上线等问题的梳理和解决方案。

### 迁移
众所周知，SpringMVC的项目有很多特有的配置文件，包括web.xml和servlet
.xml，所以迁移到SpringBoot后因为有一些配置已经默认集成到其中，而不需要迁移，还有很多需要我们自己通过SpringBoot的方式重新将其配置到项目中，一般都是通过
代码的方式，而不是配置。

#### properties
之前需要在servlet.xml中声明具体的路径和名称：
```xml
<context:property-placeholder location="classpath*:base_config.properties"/>
```
SpringBoot直接使用默认名为application.properties的文件作为配置文件(也可以用yml格式)，直接放置在 */resources/* 文件夹下即可识别。

#### Listener
SpringMVC的实现中需要在web.xml中声明引入Spring自带的请求上下文监听器：
```xml
<listener>
    <listener-class>org.springframework.web.context.request.RequestContextListener
    </listener-class>
</listener>
```
而在SpringBoot中已经完全不需要再声明这个监听器了，简化了配置。

而对于自定义的监听器，之前是使用配置文件的方式声明，这次改造为使用 *@WebListener* 注解来实现，但是需要注意在启动类的声明出使用 *@ServletComponentScan* 
来扫描类所在的包路径，否则会识别不出来。

> SpringMVC时期需要在servlet.xml文件中通过<context:component-scan base-package="com.xx"/> 标签来声明扫描包。

#### Filter
SpringMVC时期需要配置在web.xml中，并且有很多的配置项比较繁琐，如
- filter-name
- filter-class
- filter-mapping
- url-pattern
- 等等

现在可以通过使用 *@WebFilter* 注解来声明，配置参数通过注解的属性的方式配置即可，同样，需要能够被扫描到要使用 *@ServletComponentScan* 来包含对应的过滤器的包路径。

#### Interceptor
拦截器之前需要在servlet.xml文件中通过 *<mvn:interceptors/>* 标签来进行配置，可以配置多个 *bean* 。现在可以直接声明 `WebMvcConfigurer`
 接口 的实现类来替代：
```java
@Bean
WebMvcConfigurer webMvcConfigurer() {
    return new WebMvcConfigurer() {

        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(new MyInterceptor2());
            registry.addInterceptor(new MyInterceptor1());
        }

    };

}

```

#### 消息转换器
SpringMVC需要使用 *<mvc:message-converters/>* 标签来声明消息转换类型，如：
```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.ByteArrayHttpMessageConverter"/>
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter"/>
    </mvc:message-converters>
</mvc:annotation-driven>

```
而在SpringBoot中可以使用现在可以直接声明 `WebMvcConfigurer` 接口的实现类来替代：
```java
@Bean
WebMvcConfigurer webMvcConfigurer() {
    return new WebMvcConfigurer() {

        @Override
        public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
            converters.add(new ByteArrayHttpMessageConverter());
            converters.add(new MappingJackson2HttpMessageConverter());
        }
    };

}
```

#### Security
如果使用了Spring Security功能，如
```xml
<bean id="uploadFilter" class="org.springframework.security.web.FilterChainProxy">
    <security:filter-chain-map request-matcher="ant">
        <security:filter-chain
                filters="encodingFilter, requestContextFilter, uploadScopeFilter"
                pattern="/**"/>
    </security:filter-chain-map>
</bean>
```
同样可以使用编码的方式来实现：
```java
@Configuration
@EnableWebSecurity
class SpringMvcSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // disable csrf Protection because it is enabled by default in spring security
        http.cors().and().csrf().disable();
        // disable default http headers spring security sets 
        http.headers().defaultsDisabled().cacheControl();
    
        http.addFilterAfter(encodingFilter(), BasicAuthenticationFilter.class);
        http.addFilterAfter(requestContextFilter(), CharacterEncodingFilter.class);
        http.addFilterAfter(uploadScopeFilter(), RequestContextFilter.class);
    }

}
```

注意代码里面显式禁止掉了 *CORS* 、 *CSRF* 因为默认Spring Security已经支持了，如果不去掉则Reponse中可能会出现多个 
*Access-Control-Allow-Origin* 头的问题导致浏览器报错。

> The 'Access-Control-Allow-Origin' header contains multiple values '*, *', but only one is allowed.

因为一般Nginx的配置中可能已经配置了跨域的信息如：
```
add_header Access-Control-Allow-Origin "$http_origin";
add_header Access-Control-Allow-Credentials "true";
add_header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept, app-id, file-type, from-user, file-meta";
add_header Access-Control-Allow-Methods "GET, POST, OPTIONS, DELETE, PUT";
```
所以如果Nginx设置了，后端服务就可以禁用。

#### Cache Controller Header
Spring Security默认为我们增加了很多缓存控制头，其中就有：
> X-Content-Type-Options: nosniff

这个东西是啥呢？以下摘自 *JerryQu* 的[blog](https://imququ.com/post/web-security-and-response-header.html)：
```
互联网上的资源有各种类型，通常浏览器会根据响应头的Content-Type字段来分辨它们的类型。例如："text/html"代表html文档，
"image/png"是PNG图片，"text/css"是CSS样式文档。然而，有些资源的Content-Type是错的或者未定义。这时，某些浏览器会
启用MIME-sniffing来猜测该资源的类型，解析内容并执行。
```
在测试的时候发现下载资源的时候浏览器会直接将文件的数据内容展示在页面中而不是像之前一样直接下载成一个文件，原因就是Reponse中多了上面的header
使得浏览器不会去猜测下载的文件是什么类型，甚至不知道下载的是否是一个文件，所以直接将内容展示出来。

为啥浏览器会要通过 *sniff* 来判断得到的数据流的内容，原因是服务器返回数据文件内容的时候并没有设置对应MIME类型。

以下摘自[mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
```
In the absence of a MIME type, or in certain cases where browsers believe they are incorrect, browsers may perform MIME sniffing — guessing the correct MIME type by looking at the bytes of the resource.

Each browser performs MIME sniffing differently and under different circumstances. (For example, Safari will look at the file extension in the URL if the sent MIME type is unsuitable.) There are security concerns as some MIME types represent executable content. Servers can prevent MIME sniffing by sending the X-Content-Type-Options header.

```
#### Tomcat
内置容器默认使用Tomcat，我们线上服务也是使用的Tomcat，但是是8.0版本，而SpringBoot2默认的内置Tomcat已经升级到9.x版本，这里也为我们埋下了一个小坑。

需要处理的主要问题是通过代码的方式实例话Tomcat并配置相应的线程池、最大连接数以及队列和超时时间等参数，注意要和线上配置保持一致。


### 异常
#### 400错误
在线上部署时发现Nginx返回了大量的400错误，但是业务并没有log，并且看到一条奇怪的log：
```
http-nio-8080-exec-5 | DEBUG | org.apache.coyote.http11.Http11Processor(175) KEY: | The host [xxx] is not valid
java.lang.IllegalArgumentException: The character [_] is never valid in a domain name.
	at org.apache.tomcat.util.http.parser.HttpParser$DomainParseState.next(HttpParser.java:926)
	at org.apache.tomcat.util.http.parser.HttpParser.readHostDomainName(HttpParser.java:822)
	at org.apache.tomcat.util.http.parser.Host.parse(Host.java:71)
	at org.apache.tomcat.util.http.parser.Host.parse(Host.java:45)
	at org.apache.coyote.AbstractProcessor.parseHost(AbstractProcessor.java:288)
	at org.apache.coyote.http11.Http11Processor.prepareRequest(Http11Processor.java:809)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:384)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:834)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1415)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)
```

其中xxx是在Nginx中配置的upstream的变量名。

调研发现，原来是Tomcat 8.5.31/9.0.5后开启强制域名验证导致 *[xxx]* 被认为是不合法的主机名，就像堆栈里说的：

> The character [_] is never valid in a domain name.


要解决这个问题有连个思路：
- 直接将Nginx传递过来的主机名替换为真实的转发的主机名，即前端的域名
- 通过配置Tomcat的 `relaxedPathChars` 和 `relaxedQueryChars` 变量来允许现有的格式的主机名通过验证

就像Tomcat9中声明的一样：
 ```
This system property is deprecated. Use the relaxedPathChars and relaxedQueryChars attributes of the Connector instead. 
These attributes permit a wider range of characters to be configured as valid.

```

可以使用如下代码来配置：
```java

public WebServerFactoryCustomizer<TomcatServletWebServerFactory> containerCustomizer() {
    return new EmbeddedTomcatCustomizer();
}

public static class EmbeddedTomcatCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers((TomcatConnectorCustomizer) connector -> {
            connector.setAttribute("relaxedPathChars", "\"<>[\\]^`{|}");
            connector.setAttribute("relaxedQueryChars", "\"<>[\\]^`{|}");
        });
    }
}
```

```
Enable strict validation of the provided host name and port for all connectors. 
Requests with invalid host names and/or ports will be rejected with a 400 response. (markt)

```

所以该错误是Tomcat拦截到不合法的host名（即upstream名）后直接返回的。

对比Spring Cloud不同版本中不同的Tomcat版本。可见升级前版本不需严格校验，升级后版本需要。因此导致400问题。

联系运维将Nginx中增加设置Host的Header的配置信息即可：
```
location ^~ /rest/api/applyToken {
  proxy_pass http://biz_api;
  proxy-set-header    Host   $host;
```

#### javaee错误
之前是基于JAVA EE 7的，SpringBoot2.x后需要使用JAVA EE 8，否则会遇到 `NoSuchMethodError` 的异常。

在pom.xml中加入如下依赖，注意scope为 *provided* ：
```xml
<dependency>
    <groupId>javax</groupId>
    <artifactId>javaee-api</artifactId>
    <version>8.0</version>
    <scope>provided</scope>
</dependency>
```

#### Servlet异常
同样，项目需要依赖server-api，否则可能会遇到类似`NoSuchMethodError` 的异常。

在pom.xml中加入如下依赖，注意scope为 *provided* ：
```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>

```
### References
- [Spring Boot 2.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Release-Notes)
- [springboot系列文章之实现跨域请求(CORS)](https://juejin.im/post/5b99dcca6fb9a05d3154f8b7)
- [Spring Security – Cache Control Headers](https://www.baeldung.com/spring-security-cache-control-headers)

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
