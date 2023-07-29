---
layout:     post
title:      "大文件下载中的SocketTimeoutException问题定位"
subtitle:   "SocketTimeout When Downloading Big File"
date:       2020-02-28
author:     LiuShuo
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - TroubleShooting
    - Timeout
    - TCP
---
    
> 线上的文件服务器采用Tomcat+Nginx的架构部署对外提供，最近频繁遇到服务器报错SocketTimeoutException的问题，本文主要记录问题的定位过程和解决方案。

### 问题
线上的文件服务一直都有ClientAbortException的异常，但是绝大部分都是由于客户端主动断开了连接，即 *Connection reset by peer* 
，所以一直也没特别在意这个问题。
最近由于在家办公的人增多导致，不同人的网络情况不太一样，服务器相关的异常开始变多，并且有同事反馈下载某大文件时可以必现下载失败的问题。于是开始分析原因，在堆栈中发现了不同的Cause
 —— SocketTimeoutException，这个就有点意思了，于是开始顺着堆栈研究问题。

#### Nginx的角度
对于Nginx来说，它的l错误log里一直可以看到这样的异常信息：

> upstream prematurely closed connection while sending to client...

也就是说，是Tomcat关闭了连接，而不是Nginx，那Tomcat为什么会主动关闭连接呢？

#### Tomcat的角度
线上的服务架构使用的是SpringBoot2.x内嵌了Tomcat9容器，顺着堆栈找到了报错的类 *NioBlockingSelector* （来自tomcat-embed-core-9.0
.26-sources.jar中的）：
```java
/**
     * Performs a blocking write using the byte buffer for data to be written
     * If the <code>selector</code> parameter is null, then it will perform a busy write that could
     * take up a lot of CPU cycles.
     *
     * @param buf ByteBuffer - the buffer containing the data, we will write as long as <code>(buf.hasRemaining()==true)</code>
     * @param socket SocketChannel - the socket to write data to
     * @param writeTimeout long - the timeout for this write operation in milliseconds, -1 means no timeout
     * @return the number of bytes written
     * @throws EOFException if write returns -1
     * @throws SocketTimeoutException if the write times out
     * @throws IOException if an IO Exception occurs in the underlying socket logic
     */
    public int write(ByteBuffer buf, NioChannel socket, long writeTimeout)
            throws IOException {
        SelectionKey key = socket.getIOChannel().keyFor(socket.getSocketWrapper().getPoller().getSelector());
        if (key == null) {
            throw new IOException(sm.getString("nioBlockingSelector.keyNotRegistered"));
        }
        KeyReference reference = keyReferenceStack.pop();
        if (reference == null) {
            reference = new KeyReference();
        }
        NioSocketWrapper att = (NioSocketWrapper) key.attachment();
        int written = 0;
        boolean timedout = false;
        int keycount = 1; //assume we can write
        long time = System.currentTimeMillis(); //start the timeout timer
        try {
            while (!timedout && buf.hasRemaining()) {
                if (keycount > 0) { //only write if we were registered for a write
                    int cnt = socket.write(buf); //write the data
                    if (cnt == -1) {
                        throw new EOFException();
                    }
                    written += cnt;
                    if (cnt > 0) {
                        time = System.currentTimeMillis(); //reset our timeout timer
                        continue; //we successfully wrote, try again without a selector
                    }
                }
                try {
                    if (att.getWriteLatch() == null || att.getWriteLatch().getCount() == 0) {
                        att.startWriteLatch(1);
                    }
                    poller.add(att, SelectionKey.OP_WRITE, reference);
                    att.awaitWriteLatch(AbstractEndpoint.toTimeout(writeTimeout), TimeUnit.MILLISECONDS);
                } catch (InterruptedException ignore) {
                    // Ignore
                }
                if (att.getWriteLatch() != null && att.getWriteLatch().getCount() > 0) {
                    //we got interrupted, but we haven't received notification from the poller.
                    keycount = 0;
                } else {
                    //latch countdown has happened
                    keycount = 1;
                    att.resetWriteLatch();
                }

                if (writeTimeout > 0 && (keycount == 0)) {
                    timedout = (System.currentTimeMillis() - time) >= writeTimeout;
                }
            }
            if (timedout) {
                throw new SocketTimeoutException();
            }
        } finally {
            poller.remove(att, SelectionKey.OP_WRITE);
            if (timedout && reference.key != null) {
                poller.cancelKey(reference.key);
            }
            reference.key = null;
            keyReferenceStack.push(reference);
        }
        return written;
    }

```
当超时判断字段为true时抛出了 *SocketTimeoutException* 异常，从代码里可以看到，如果写顺利，每次都会更新 *time* 字段，理论上不会使得 * timeout* 
为true。所以，一定是写回给Nginx的时候遇到了一些问题，导致很长时间没有更新 *time* 字段并最终超过了配置的超时时间导致的报错。

OK，基本思路已经清晰，那么是什么参数来控制这个超时阈值呢？是否可以改变呢？ *write()* 方法里的参数 *writeTimeout* 又是多少呢？

经过对Tomcat内置参数的修改以及debug，最终确定影响上述时间参数的配置项是 *connectionTimeout* 而不是 *soTimeout* ，这个和Tomcat的官方文档中
对该字段的解释不是特别的匹配，一直以为这个字段是用来控制建立连接的时长的，这个寻找过程也花了不少时间，因为从*soTimeout*、 *socketTimeout* 和 
*keepAliveTimeout* 这几个参数中一直调试上线观测都没什么效果。

通过调大 *connectionTimeout* 参数发现线上报错时统计的耗时都会大于这个值（这也是为什么确定是这个参数对结果有影响的原因），所以这个思路可以一定程度解决这个问题。

问题是设置多大呢？设置大了会不会对服务器有影响呢？

答案是肯定的，如果设置的太大，在有大文件并发下载时有可能会遇到服务器内存压力升高、GC耗时和CPU耗时增大的风险，所以不能一味的调高这个值，需要根据实际情况调整。
最终我调整为60s后没再调高，因为服务器的异常情况已经基本绝迹了。

### 原因分析
#### 理解超时
如果对 *connectionTimeout* 的认识不够清楚则可能会就认为是时间太短导致大量数据没有「写完」导致的报错，尤其是默认我们使用的是2s就会更加倾向于这样的答案。在上一篇文章中
我分析了关于Tomcat和Nginx的一些超时参数的含义，不难了解，这里的2s如果在网络环境足够好的情况下，不管传多大的文件都不会有问题。

那么问题出自哪里呢？

是客户端的网络太差，导致会有丢包和延迟，而Tomcat写数据是基于TCP的滑动窗口机制，只有当对端接收到并返回ACK后发送端才会将这部分数据从窗口中移动出去（其实是
窗口向前滚动），如果一直没有收到，即「Send But Not Yet Acknowledged」，则会在超时时间达到后抛出异常。

#### 跟Nginx没有关系么？
有，但关系不大。Nginx主要是负责转发Client和Tomcat的数据给对方，虽然它也会做一些本地缓存如proxy_buffering等，我们的配置默认没有开，这样的好处是
Nginx在收到数据之后会立刻写给Client而不会进行缓存，这样能够达到最小的延迟。如果开启了buffering机制和增大buffer_size只会增大Nginx的内存消耗以及增加延时，
如果是客户端网络的原因导致回包不及时，是爱莫能助的。

同时为了加快数据尽快的发送到客户端，Nginx可以开启TCP_NODELAY选项，这样就可以使缓冲区中的数据立刻发出去，这样带来的一个问题就是Tomcat每产生一个包就会被发送出去，
如果按照一个包拥有一个死结的数据和40个字节的包头（IP数据包包头的大小20Bytes和TCP数据段的包头20Bytes）来算，则产生了4000%的过载，可能会造成网络拥堵，降低带宽利用率并增加了延迟。

> TCP_NODELAY 启用后会禁用 Nagle 算法，尽快发送数据，某些情况下可以节约 200ms（Nagle 算法原理是：在发出去的数据还未被确认之前，新生成的小数据先存起来，凑满一个 
MSS（Maximum Segment Size，1460Bytes=1500 - IP头(20) - TCP头(20)） 或者等到收到确认后再发送）。Nginx 只会针对处于 keep-alive 状态的 TCP 连接才会启用。

对于Nginx来说，需要保证它的proxy_connection_timeout/proxy_read_timeout/proxy_write_timeout
足够大，不至于因为自己配置的原因影响后端Tomcat的数据读写，做好自身转发的工作即可。

#### 增大超时时间为啥可以解决问题
增加超时时间，给因为IP网络丢包原因导致的异常情况以更多的时间进行丢包重传，所以可以缓解Socket超时这个问题。

### References
- [系统调优你所不知道的TIME_WAIT和CLOSE_WAIT](https://zhuanlan.zhihu.com/p/40013724)
- [Nginx与Tomcat性能调优，前后端KeepAlive不同步引发的问题](https://blog.csdn.net/nimasike/article/details/81129163)

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
