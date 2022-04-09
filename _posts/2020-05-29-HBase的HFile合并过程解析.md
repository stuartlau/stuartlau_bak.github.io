---
layout:     post
title:      "HBase的HFile合并过程解析"
subtitle:   "HFile Compaction in HBase"
date:       2020-05-29
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - HBase
---

> Compaction是指一个region 的一个 store 中的多个 HFile 合为一个 HFile 的操作，HBase在执行合并时会对上层业务产生比较大的影响。
本文对其中的执行原理和过程进行一定的研究。
  
compaction在以LSM-Tree为架构的系统中是非常关键的模块，log append的方式带来了高吞吐的写，内存中的数据到达上限后不断刷盘，
数据范围互相交叠的层越来越多，相同key的数据不断积累，引起读性能下降和空间膨胀。因此，compaction机制被引入，
通过周期性的后台任务不断的回收旧版本数据和将多层合并方式来优化读性能和空间问题。
![](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig19.png)
### 合并storefile的原因
我们知道HBase的数据默认会先写到HLog中，然后写到memstore中，当memstore写成功后就立刻返回给客户端写回成功。

当数据越来越多直到memstore占满，会将其刷新到硬盘的storefile中，
并且每次写入形成一个单独storefile，当storefile达到一定的数量后，如果不进行合并处理，会降低查询的效率，因为Hadoop
不擅长处理小文件，文件越大性能越好。

### 合并分类
HBase 根据合并规模将 Compaction 分为了两类：
- Minor Compaction是指选取一些`小的、相邻的`StoreFile将他们合并成一个更大的StoreFile，在这个过程中不会处理已经Deleted
或Expired的Cell。一次Minor Compaction的结果是更少并且更大的StoreFile。

- Major Compaction是指将`所有的`StoreFile合并成一个StoreFile，这个过程还会清理三类无意义数据：被删除的数据、TTL
过期数据、版本号超过设定版本号的数据。另外，一般情况下，Major Compaction时间会持续比较长，整个过程会消耗大量系统资源，
对上层业务有比较大的影响，比如PUT操作耗时会增加。因此线上业务都会将关闭自动触发Major Compaction功能，改为手动在业务低峰期触发。

![](https://uzshare.com/_p?https://img-blog.csdnimg.cn/20190528091325628.png#pic_center)
### 合并时机
触发compaction的方式有三种：Memstore刷盘、后台线程周期性检查、手动触发。

1.Memstore Flush:

应该说compaction操作的源头就来自flush操作，Memstore Flush会产生HFile文件，文件越来越多就需要compact。
因此在每次执行完Flush操作之后，都会对当前Store中的文件数进行判断，一旦文件数大于配置，就会触发compaction。
需要说明的是，compaction都是以Store为单位进行的，而在Flush触发条件下，整个Region的所有Store都会执行compact，
所以会在短时间内执行多次compaction。

2.后台线程周期性检查：

后台线程定期触发检查是否需要执行compaction，检查周期可配置。线程先检查文件数是否大于配置，一旦大于就会触发compaction。
如果不满足，它会接着检查是否满足major compaction条件，简单来说，如果当前Store中HFile的*最早更新时间*早于某个值mcTime，
就会触发major compaction（默认7天触发一次，可配置手动触发），HBase预想通过这种机制定期删除过期数据。

3.手动触发：

一般来讲，手动触发compaction通常是为了执行Major Compaction，一般有这些情况需要手动触发合并是因为很多业务担心
自动Major Compaction影响读写性能，因此会选择低峰期手动触发；

也有可能是用户在执行完alter操作之后希望立刻生效，执行手动触发Major Compaction；或HBase管理员发现硬盘容量不够的情况
下手动触发Major Compaction删除大量过期数据；

### 如何实现合并后的排序
由于内存里memstore是在数据插入的过程中就排序的，就是数据插入的时候按照顺序插入，所以memstore里的数据是有序的。
当memstore的数据刷写到磁盘时，生成的storefile里的数据也是有序的，这样的话各个storefile里的数据就分别有序了。
合并的时候需要将各个有序的storefile合并成一个大的有序的storefile。

首先将各个需要合并的storefile封装成`StoreFileScanner`，最后形成一个List加载到内存，然后再封装成`StoreScanner`
对象，这个对象初始化的时候会对各个`StoreFileScanner`进行排序放到内部的队列里，排序是按照各个`StoreFileScanner`最小
的rowkey进行排序的。然后通过StoreScanner的next()方法可以拿到各个`StoreFileScanner`最小rowkey中的最小rowkey对应的KV对。
然后就把取出的KV对追加写入合并后的storefile。因为每次取出的都是各个storefile里最小的数据，所以追加写入合并后的storefile里的数据就是按从小到大排序的有序数据。

### 合并时的操作
在合并的过程中会抛弃删除标识的行和版本过旧的行
-（1）可以预先定义版本的个数，超过这个值就抛弃
-（2）还可以预先定义版本的时间长短，超过这个时间就抛弃，合并完后形成更大的storefile，当达到数量再次合并，
直到storefile容量超过一定阀值后会把当前的Region进行分裂为2个并由HMaster（Hbase数据库主控节点）
分配到不同的HRegionServer服务器处理实现负载均衡。

### 合并时如何处理查询请求
如果在合并过程中恰好有涉及到有关storefile的查询发生的话，先是把小storefile加载到内存中进行合并此时如有用户访问可以在
内存中检索相关数据返回给用户，我们可以想象在内存中做一个独立镜像备份专门提供被查询需求，另一个主体在另一块内存空间里进行合并，
当合并完成后释放备份的内存空间，返回到原来的状态。

### Compaction 对读写请求的影响
- 存储上的写放大

![](https://uzshare.com/_p?https://img-blog.csdnimg.cn/20190528091806822.png#pic_center)
随着minor compaction以及major Compaction的发生，可以看到，这条数据被反复读取/写入了多次，这是导致写放大的一个关键原因，这里的写放大，涉及到网络IO与磁盘IO，因为数据在HDFS中默认有三个副本。

- 读路径上的延时毛刺

HBase执行compaction操作结果会使文件数基本稳定，进而IO Seek次数相对稳定，延迟就会稳定在一定范围。然而，
compaction操作会带来很大的带宽压力以及短时间IO压力。因此compaction就是使用短时间的IO消耗以及带宽消耗换取后续查询的低延迟。
这种短时间的压力就会造成读请求在延时上会有比较大的毛刺。下图是一张示意图，可见读请求延时有很大毛刺，但是总体趋势基本稳定。
![](https://uzshare.com/_p?https://img-blog.csdnimg.cn/20190528091852202.png#pic_center)

- 写请求上的短暂阻塞

Compaction对写请求也会有比较大的影响。主要体现在HFile比较多的场景下，HBase会限制写请求的速度。
如果底层HFile数量超过`hbase.hstore.blockingStoreFiles` 配置值，默认10，flush操作将会受到阻塞，
阻塞时间为`hbase.hstore.blockingWaitTime`，默认90000，即1.5分钟，在这段时间内，
如果compaction操作使得HFile下降到blockingStoreFiles配置值，则停止阻塞。另外阻塞超过时间后，
也会恢复执行flush操作。这样做可以有效地控制大量写请求的速度，但同时这也是影响写请求速度的主要原因之一。


### Reference
- [HBase篇（6）-HFile合并过程详解](https://cloud.tencent.com/developer/news/366464)
- [深入探讨LSM Compaction机制](https://zhuanlan.zhihu.com/p/141186118)
- [深入理解 HBase Compaction 机制](https://uzshare.com/view/788152)

> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
