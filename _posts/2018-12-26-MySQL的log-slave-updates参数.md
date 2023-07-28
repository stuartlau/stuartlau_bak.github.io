---
layout:     post
title:      MySQL的log-slave-updates参数
subtitle:   MySQL with log-slave-updates 
date:       2018-12-26
author:     L.S
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - MySQL
---
> 为什么讨论这个问题，我参与的一个业务的MySQL集群的拓扑图是一条线，跟我之前的理解不太一致，于是研究了一下这样做的好处，然后又研究了一下如何才能做成这样的拓扑，首先遇到了log-slave
-updates...

## 链式拓扑
线上的一个业务申请了MySQL数据库集群，双机房部署，主机房和从机房，写实例在主机房，单库，其他均为Slave。
拓扑图如下：
> M<---S1<---S2<---S3<---S4<---S5

每个机房三台机器，前三个为主机房的机器，后三个为从机房的机器，除了M之外都是Slave，并且从机房的第一台机器S3是主机房拓扑里第三台机器S2的Slave。
除了M之外其他所有机器都是read-only的，并且除了最后一个S5之外，所有机器都有一个replica。
也就是说在上面这个线性拓扑图中，后面的机器依赖直接父节点机器的同步进度来更新自己的数据，在延迟较高时，瓶颈越靠前，影响的实例越多。

### 链式优点
- binlog下游消费：Cache数据必须可回查
- 切换逻辑简单：快速择主，切换可预知
- 灾备机房切换：在短时间内剔除线上全部某一个机房的实例

### 链式缺点
- DB自身复制延时放大
- Crash场景数据丢失风险增大

## 参数定义
官方的解释如下：
> Normally, a slave does not log to its own binary log any updates that are received from a 
master server. This option tells the slave to log the updates performed by its SQL thread to its 
own binary log. For this option to have any effect, the slave must also be started with the --log-bin option to enable binary logging. Prior to MySQL 5.5, the server would not start when using the --log-slave-updates option without also starting the server with the --log-bin option, and would fail with an error; in MySQL 5.5, only a warning is generated. (Bug #44663) --log-slave-updates is used when you want to chain replication servers. For example, you might want to set up replication servers using this arrangement:
>  
>  A -> B -> C
>  
>  Here, A serves as the master for the slave B, and B serves as the master for the slave C. For 
this to work, B must be both a master and a slave. You must start both A and B with --log-bin to enable binary logging, and B with the --log-slave-updates option so that updates received from A are logged by B to its binary log.

可知，如果一台服务器同时作为slave和master则需要开启这个参数，否则Slave只会更新自己是Relay Log并不会写Bin Log。

### 拓扑类型
一般情况下MySQL的replica可以有以下几种架构：
- Master-Slaves Mode
    - 一个M和可以有N个S，彼此S之间不通讯
    - 适合读多写少的情况，N太大对M的负载和网络带宽有较大压力   
    - 不同的slave扮演不同的作用(例如使用不同的索引，或者不同的存储引擎)
    - 用一个slave作为备用master，只进行复制（不提供读取）
    - 用一个远程的slave，用于灾难恢复（即多机房）

- Master-Master in Active-Active Mode
    - Master-Master复制的两台服务器，既是master，又是另一台服务器的slave
    - 并发更新会有问题，如同一个数据在两边执行不同的SQL后结果不同
        > 假设一个表只有一行(一列)的数据，其值为1，如果两个服务器分别同时执行如下语句：
        >
        > 在第一个服务器上执行：
        >
        >  mysql> UPDATE tbl SET col=col + 1;
        >
        >  在第二个服务器上执行：
        >
        >  mysql> UPDATE tbl SET col=col * 2;
        >
        >  结果将是：一台服务器是4，另一个服务器是3
- Master-Master in Active-Passive Mode
    - 由master-master结构变化而来的
    - 避免了M-M的缺点
    - 具有容错和高可用性
    - 其中一个服务只能进行只读操作

- Master-Master with Slaves
    - 这种结构的优点就是提供了冗余
    - 在地理上分布的复制结构
    - 不存在单一节点故障问题
    - 可以将读密集型的请求放到slave上

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
