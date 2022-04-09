---
layout:     post
title:      "关于Nginx和Tomcat的Timeout相关配置的分析"
subtitle:   "Timeouts in Nginx and Tomcat"
date:       2020-02-26
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Web
    - Nginx
    - Tomcat
---
    
> 本文主要梳理在Web开发时遇到的各种超时问题的总结，主要包括熟悉的Tomcat和Nginx的一些常用配置的解答和测试。

### Tomcat
#### connectionTimeout
先来看看[官方文档](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html)的解释：
> connectionTimeout
>  
> The number of milliseconds this Connector will wait, after accepting a connection, for the 
request URI line to be presented. Use a value of -1 to indicate no (i.e. infinite) timeout. The default value is 60000 (i.e. 60 seconds) but note that the standard server.xml that ships with Tomcat sets this to 20000 (i.e. 20 seconds). Unless disableUploadTimeout is set to false, this timeout will also be used when reading the request body (if any).

理解起来似乎并不难，但是它有什么作用呢？

在[stackoverflow](https://stackoverflow.com/a/59621460/1842018)找到一个比较好的解答：
> This parameter is there specifically to fight one type of Denial-Of-Service attack, whereby some malicious client(s) create a TCP connection to the server (which has theeffect of reserving some resources on the server for handling this connection), and then just sit there without sending any HTTP request on that connection. By making this delay shorter, you shorten the time during which the server resources are allocated, to serve a request that will never come.

也就说这个参数主要用于控制和客户端之间的连接的超时时间，也是防止Denial-Of-Service攻击的有效手段，有些恶意的客户端在建立完TCP连接之后不发送任何HTTP请求，服务器
如果不对这种行为进行有效管控，则很快就会消耗完所有线程池资源，无法进行服务，该参数就是用来控制连接建立后在多长时间内服务器可以主动关闭连接的。


如果想对POST情况不使用connectionTimeout来限制，还有另外两个参数可用。这两个参数必须配合使用才行：
```
disableUploadTimeout="false"
connectionUploadTimeout="10000"
```

必须要设置disableUploadTimeout为false（默认是true），才可以对POST请求发送数据超时使用其他参数来设置，这样在发送数据的过程中最大可以等待的时间间隔就不再由connectionTimeout决定，而是由connectionUploadTimeout决定。

做个试验，我用SpringBoot启动一个内嵌Tomcat9的Web服务，并设置对应的connectionTimeout为2s，通过telnet可以模拟客户端的恶意行为：
- 连接后不发任何信息

```
$ time telnet localhost 8080
Trying ::1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.

real	0m2.037s
user	0m0.001s
sys	0m0.001s
```

可以看到经过了大概2s的时间后连接被服务器断了。

- 连接后发送POST请求并不发送完整数据

```
$ time telnet localhost 8080
Trying ::1...
Connected to localhost.
Escape character is '^]'.
POST /test HTTP/1.1
host: localhost:8080
Content-type:application/x-www-form-urlencoded
Content-length:10
2Connection closed by foreign host.

real	0m5.383s
user	0m0.001s
sys	0m0.001s
```

可以发现这次延长到了5s才被服务器关闭，是因为我们增加了一些HTTP报文的传输，但是在输入内容的时候又停住了，等待2s后又被服务器关闭了。

还可以通过一直发送回车的方式保持和服务器的连接，请自行测试。


### Nginx
#### proxy_connect_timeout
> Defines a timeout for establishing a connection with a proxied server. It should be noted that this timeout cannot usually exceed 75 seconds.
  
在收到请求头后，会将请求转发到 *upstream* 里面配置的backend server，这个就是与对应的server连接的超时时间，设置时最大值不能超过75s，
比如我们使用Tomcat和Nginx是放在同一个交换机上的内网，所以将连接时间优化到10s，超过10s连接不上，说明业务有问题了。


#### proxy_read_timeout

> Defines a timeout for reading a response from the proxied server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the proxied server does not transmit anything within this time, the connection is closed.
  
注意该指令并不是定义Client从Server
读取数据的耗时时间，而是在两个连续的读操作之间的时间间隔，即有回包之后的下一次回包之间的时间差，如果因为网络原因或者丢包或者速度慢可能会造成两个连续的ACK收到的时间间隔超过这个时间，
这个时候Nginx就会断开这个连接了。不过一般Nginx可能还没到这个时间就被proxied server给断掉了，这个需要协调两边的超时时间配置。

#### proxy_send_timeout
> Sets a timeout for transmitting a request to the proxied server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the proxied server does not receive anything within this time, the connection is closed.
  
同样，写超时和读超时的意义是一样的，都是用来定义两个连续的写操作之间的时间间隔的最大值，否则会断掉和proxied server的连接。

### Conclusion
熟悉Tomcat和Nginx的一些超时的配置对我们处理日常的一些网络超时等错误有非常大的帮助，本文会不断更新对这些看起来你懂但其实就是用不好的配置和参数的解读。

### References
- [tomcat-connector的微调(4): 超时相关的参数](http://hongjiang.info/tomcat-connector-tuning-4/)
- [Nginx Documentation - HTTP Proxy Module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)

> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
