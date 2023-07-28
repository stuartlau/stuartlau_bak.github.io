---
layout:     post
title:      使用Wireshark抓取Redis协议包
subtitle:   你得做一个合格的工程师
date:       2018-04-10
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Redis
    - Wireshark
---
# 测试一

![wireshark-view-0](http://stuartlau.github.io/img/in-post/wireshark-view-0.jpg)

本地连接服务器Redis服务，保持了很久，确定连接超过超时时间后再次发送info命令，发现:
- 1.原来的端口连接被重置了，服务器发送RST，且win=0，len=0
- 2.客户端重新发起三次握手建立连接，端口为50201，注意握手的时候的seq=0，对方一定ack回1，尽管len=0，MSS=1460表示最大的包的大小Max Segment 
- Size，并商量了TSval和TSce
- 3.然后立即重发info命令，seq和ack与之前的相同，len=14字节，报告本地滑动窗口大小
- 4.服务器收到后发送了3个segment，第一个告诉客户端我收到了回了个ack；第二个发送了一部分info的返回内容，因为超过了MSS所以被拆分了两个回包
- 5.客户端收到后回一个ack包，len=0即没有数据发送

# 测试二
虽然一段时间没有交互则服务器会主动发一个ack keep-alive来探测客户端的连通性，其中seq比之前的小1，客户端收到会回复ack并且自动加1，即与正常交互时最后一个seq相同

# 测试三

![wireshark-view-2](http://stuartlau.github.io/img/in-post/wireshark-view-2.jpg)

- 1.客户端发送get “A” 
- 2.服务端cluster回复MOVED 10.50.2.16 7003，即表示这个key的slot不在7001这里而在7003那里
- 3.客户端回包ack
- 4.客户端发起FIN ACK断链操作，7001回复FIN ACK，客户端回复ACK，注意断链的时候的ack都是默认加对方seq的1，即使len=0
- 5.建立和7003的连接，跟之前的一样，先是三次握手，然后发命令收响应

# 测试四

![wireshark-view-3](http://stuartlau.github.io/img/in-post/wireshark-view-3.jpg)
- 1.发exit命令断开连接
- 2.客户端netstat -an|grep 7003
tcp4       0      0  192.168.124.79.50405   10.50.2.16.7003        TIME_WAIT
- 3.状态会显示一段时间，等2MSL后才会消失


> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 转载请保留原文链接.
