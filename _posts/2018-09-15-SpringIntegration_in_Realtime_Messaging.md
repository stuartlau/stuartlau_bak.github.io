---
layout:     post
title:      Spring Integration in Realtime Messaging
subtitle:   Spring Integration and its usage in realtime messaging system
date:       2018-09-15
author:     S.L
header-img: img/post-bg-spring-integration.png
catalog: true
tags:
    - Messaging
    - Spring
---

### Table of Content
* Realtime Submission Messaging Brief
* Old Design with Self-Implemented Framework
* Spring Integration Introduction
* New Design with Spring Integration



## Realtime Submission Messaging Brief
* Message consuming, e.g. from AWS Kinesis
* Different logic for different markdown actions, e.g. CREATE, UPDATE, RETRACT
* Filtering, e.g. unsupported Realm, invalid data
* Multi-Threading, lots of Producer&Consumer scenarios
* Dependent on other services getting required data based on different data type, e.g. SDVP data, config data
* Logics to compute core business values, e.g. Duration, Qty
* A classic messaging process, consuming data -> validity checking -> filter out invalid -> splitting -> routing to different handlers -> different execution -> publishing

## Old Design with Self-Implemented Framework
* All the logics such as blocking queues(e.g. *ArrayListBlockingQueue*), multiple threading(e.g. *ExecutorService*), pub-sub are self-implemented, wasting time in writing non-business UTs, e.g. queue functions
* All the input collection logics using *Chain Strategy*, not easy to get a quick view and painful to change structure
* Filter logic resides in the heart of business logic, not SRP
* Some logics are roughly coupled or don't know where exactly it should be, e.g. Splitting resides with message receiving logic
* Exception handling is everywhere and hard to extract as extensible component
* Temp values newed everywhere, not clean code
* Logging is heavily coupled with business handlers
* Add new code when data is desired by other component 
* High maintenance
* Boundary between components is vague


## Spring Integration Introduction
> Spring Integration's primary goal is to provide a simple model for building enterprise integration solutions while maintaining the separation of concerns that is essential for producing maintainable, testable code.

`### Brief
* Enables lightweight messaging within Spring-based applications
* Supports integration with external systems via declarative adapters(File, FTP, JMS, TCP, HTTP, JDBC, etc)
* Provides a higher-level of abstraction over Spring's support for remoting, messaging, and scheduling

Spring Integration is motivated by the following goals: 

* Provide a simple model for implementing complex enterprise integration solutions
* Facilitate asynchronous, message-driven behavior within a Spring-based application
* Promote intuitive, incremental adoption for existing Spring users

Spring Integration is guided by the following principles: 

* Components should be loosely coupled for modularity and testability
* The framework should enforce separation of concerns between business logic and integration logic
* Extension points should be abstract in nature but within well-defined boundaries to promote reuse and portability



### Basic Component
* Message
* Channel
* Filter
* Splitter
* Router
* Transformer
* Activator
* Chain
* Gateway


#### 1.Message

![Message](http://docs.spring.io/spring-integration/reference/htmlsingle/images/message.jpg)

* Comprised of *Payload* and *Headers*
* *Payload* is your business Object
* *Header* takes additional K-V data, like *Context* in Web development which takes IP, SessionId, or fields like cc, bcc, title in an Email scenario that don't belong to text body
* PreDefined fields: *UUID*, *Timestamp*, *ReplyChannel*, *CorrelationId*

#### 2.Channel
* *Queue* that holds *Message*
* Decouple producer and consumer, no need to invoke put() or get(), all encapsulated internally
* *Point2Point*&*PubSub* 
* *Inbound*&*Outbound*
![](http://docs.spring.io/spring-integration/reference/htmlsingle/images/channel.jpg)
* *Pollable* Channel, enabling buffering and sheduling
* *Priority* Channel, using Comparator
* *Subscribable* Channel, message driven
* DataType setting supported, no chance for messy data
* Interceptor supported, e.g. `preSend`, `preReceive`, `afterReceiveCompletion`, etc. Usage: `logging`, `counting`, etc.
* Wire-tap *Message* to another *Channel*(same data)
e.g. you can make the *Messages* in a *Channel* also flow to another one at the same time


```xml
<int:channel id="submissionChannel" datatype="io.github.stuartlau.si.model
.InputWrapper">
        <int:interceptors>
            <int:wire-tap channel="loggerChannel"/>
        </int:interceptors>
</int:channel>
```
e.g. or you can use a wire-tap with a pattern, no need to configure inside a channel

```xml
<int:wire-tap pattern="*ActionChannel, markdownRequestChannel" order="0" channel="metricsWiretapChannel"/>
```



#### 3.Filter
* Filter out invalid *Messages*, lessen the load of backend service
* Ignore invalid *Messages* using exception or throw them using another *Channel*, depends on your business
* Exception *Channel* for *Subscribers* to consume



e.g. filter invalid *Message* and throw them into *filterErrorChannel*

```xml
<int:channel id="filterErrorChannel" />

<int:filter ref="markdownRequestFilter" throw-exception-on-rejection="false" discard-channel="filterErrorChannel" />
```



#### 4.Splitter
* Split composition message to component messages
* From Object to a collection of Objects

e.g. *splitter* splits *Message* from *markdownRequestChannel1* and put the pieces to *markdownRequestChannel2*

```xml
<int:splitter id="splitter" ref="markdownRequestSplitter" input-channel="markdownRequestChannel1" output-channel="markdownRequestChannel2/>
```
#### 5.Router
![Router](http://docs.spring.io/spring-integration/reference/htmlsingle/images/router.jpg)

* Route different messages in a *Channel* to different *Channels*
* Different *Channels* consumed by different *Subscribers*, like *Strategy Pattern*

e.g. route messages in *submissionChannel* to different channels using self-defined strategy, and default-output-channel could consume no-routing ones, no one is ignored

XML

```xml
<int:router input-channel="submissionChannel" ref="markdownActionRouter" default-output-channel="defaultRouterChannel" />
```

Interface

```java
public String route(InputWrapper inputWrapper) {
    String channelName;
    // choose with channel this message should be sent to
    // return the name of the channel
	return channelName;
}
```


#### 6.Transformer
* Change the content of the *Payload* of *Message*, e.g. add, edit or delete values
* Change to another type of Payload type, e.g. BatchelorBean to MasterBean, MasterBean to DoctorBean

e.g. add *InventoryPicture* data into the *Payload* of *Message*

XML

```
<int:transformer id="inventoryPicTransformer" ref="inventoryPicTransformer"/>
```

Interface


```java
public InputWrapper transform(Message<InputWrapper> message) {
		InputWrapper inputWrapper = message.getPayload();
        GetDataByFnskuScopeResponse response = sdvpServiceInvoker.getInventoryInfoData(inputWrapper);
        extractInventoryInfoData(inputWrapper, response);
        return inputWrapper;
}

```




#### 7.Activator
* Endpoint to handle *Message*
* *Inbound* Channel must be provided
* Outsourcing behavior can be controlled by *Outbound* Channel if provided, or else no output

e.g an *Activator* consuming a channel and output the result to another channel


```xml
<int:service-activator input-channel="createActionChannel" output-channel="publishChannel" ref="createMarkdownProcessor" >
</int:service-activator>
```

e.g. no output at all

```xml
<int:service-activator input-channel="metricsWiretapChannel" ref="monitorActivator" />
```


#### 8.Chain
* Combine the above components as a big entity that work together like a flow
* Single Inbound Channel and OutBound Channel for the whole components
* All components in a *Chain* are executed in a sequential order

* *Chain* could be connected with each other with *Channel*

e.g. a chain to process InputData(from the very beginning data to the final business data) and output to another channel for its subscribers to consume

```xml
<int:chain id="inputCollectionSubmissionChain" input-channel="inputCollectionSubmissionChannel" output-channel="submissionChannel">
        <int:poller task-executor="inputCollectionExecutor" fixed-rate="1" error-channel="inputErrorChannel"/>
        <int:header-enricher>
            <int:header name="request" ref="markdownRequestEnricher" method="getRequest" />
            <int:header name="startDate" ref="startDateEnricher" method="getProcessStartDate" />
        </int:header-enricher>
        <int:transformer id="submissionInputWrapperTransformer" ref="submissionInputWrapperTransformer"/>
        <int:transformer id="quantityTransformer" ref="quantityTransformer"/>
        <int:transformer id="durationTransformer" ref="durationTransformer"/>
</int:chain>
```

#### 9.Gateway
* Place *Message* into a *Channel* and kick off the rest of Spring Integration components
* Dependency to insourcing service

XML

```xml
<int:gateway id="messagePlacer" service-interface="io.github.stuartlau.si.gateway.MessagePlacer" default-request-channel="markdownRequestChannel" />
```

Interface

```java
public interface MessagePlacer {
    void place(MarkdownRequest request);
}
```


## New Design Using Spring Integration
* Internal queues are out of the box, no need to implement the basic stuff and focus on business
* More flexible to test with single or multiple threads
* Interceptors are everything, logging, monitoring, debugging
* Easy to test each component, no coupling, the gluing is under the flow
* Construct structures using XML is more readable and deployment friendly
* A standard framework focusing on message processing: routing, filtering, splitting, transforming, aggregating and exception handling can be achieved with less code(and UT) and has more extensibility

## Messaging Flow

![Messaging_Flow](http://stuartlau.github.io/img/in-post/messaging_flow.jpg)

## Pro and Con
* When your problem is broken into lots of small ones, and the boundaries are clear, you need a 
standard way to manage them efficiently, use SI
* When you want other developers get a better and quick whole picture of your complicated message processing system, use SI

* When your problem is not complicated enough, just implement with your own framework




## Reference
* [http://projects.spring.io/spring-integration/](http://projects.spring.io/spring-integration/)
* [https://github.com/spring-projects/spring-integration-samples/](https://github.com/spring-projects/spring-integration-samples/)
* [https://docs.spring.io/spring-integration/docs/](https://docs.spring.io/spring-integration/docs/)
* [http://www.eaipatterns.com](http://www.eaipatterns.com)
* [http://camel.apache.org/](http://camel.apache.org/)
* [https://www.mulesoft.com](https://www.mulesoft.com)

> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
