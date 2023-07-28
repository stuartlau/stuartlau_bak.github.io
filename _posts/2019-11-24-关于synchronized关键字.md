---
layout:     post
title:      "关于synchronized关键字"
subtitle:   "synchronized in Java"
date:       2019-11-24
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - Synchronization
---
    
> synchronized是Java中重要的同步锁工具，本文对其原理进行分析。

### Monitor Object设计模式
我们在开发并发的应用时，经常需要设计这样的对象，该对象的方法会在多线程的环境下被调用，而这些方法的执行都会改变该对象本身的状态。
为了防止竞争条件 (race condition) 的出现，对于这类对象的设计，需要考虑解决以下问题：
- 在任一时间内，只有唯一的公共的成员方法，被唯一的线程所执行。
- 对于对象的调用者来说，如果总是需要在调用方法之前进行拿锁，而在调用方法之后进行放锁，这将会使并发应用编程变得更加困难。
- 如果一个对象的方法执行过程中，由于某些条件不能满足而阻塞，应该允许其它的客户端线程的方法调用可以访问该对象。

我们使用 Monitor Object 设计模式来解决这类问题：将被客户线程并发访问的对象定义为一个 monitor 对象。
客户线程仅能通过 monitor 对象的同步方法才能使用 monitor 对象定义的服务。为了防止陷入竞争条件，在任一时刻只能有一个同步方法被执行。
每一个 monitor 对象包含一个 monitor 锁，被同步方法用于串行访问对象的行为和状态。
此外，同步方法可以根据一个或多个与 monitor 对象相关的 monitor conditions 来决定在何种情况下挂起或恢复他们的执行。
#### 结构
在 Monitor Object 模式中，主要有四种类型的参与者：

- 监视者对象 (Monitor Object): 负责定义公共的接口方法，这些公共的接口方法会在多线程的环境下被调用执行。
- 同步方法：这些方法是监视者对象所定义。为了防止竞争条件，无论是否同时有多个线程并发调用同步方法，还是监视者对象含有多个同步方法，在任一时间内只有监视者对象的一个同步方法能够被执行。
- 监视锁 (Monitor Lock): 每一个监视者对象都会拥有一把监视锁。
- 监视条件 (Monitor Condition): 同步方法使用监视锁和监视条件来决定方法是否需要阻塞或重新执行。
#### 执行序列图
- 1、同步方法的调用和串行化。当客户线程调用监视者对象的同步方法时，必须首先获取它的监视锁。只要该监视者对象有其他同步方法正在被执行，获取操作便不会成功。在这种情况下，客户线程将被阻塞直到它获取监视锁。当客户线程成功获取监视锁后，进入临界区，执行方法实现的服务。一旦同步方法完成执行，监视锁会被自动释放，目的是使其他客户线程有机会调用执行该监视者对象的同步方法。
- 2、同步方法线程挂起。如果调用同步方法的客户线程必须被阻塞或是有其他原因不能立刻进行，它能够在一个监视条件上等待，这将导致该客户线程暂时释放监视锁，并被挂起在监视条件上。
- 3、监视条件通知。一个客户线程能够通知一个监视条件，目的是为了让一个前期使自己挂起在一个监视条件上的同步方法线程恢复运行。
- 4、同步方法线程恢复。一旦一个早先被挂起在监视条件上的同步方法线程获取通知，它将继续在最初的等待监视条件的点上执行。在被通知线程被允许恢复执行同步方法之前，监视锁将自动被获取。

下图中描述了监视者对象的动态特性：
![monitor_object](https://img-blog.csdn.net/20160723160011822)

### 理解Java对象头与Monitor
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。如下：
- 对象头：Java头对象，它实现synchronized的锁对象的基础
- 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
- 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。


一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成，其结构说明如下表：

|虚拟机位数|头对象结构|说明|
|32/64bit|Mark Word|存储对象的hashCode、锁信息或分代年龄或GC标志等信息|
|32/64bit|Class Metadata Address|类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例。|

其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

|锁状态|25bit	|4bit|1bit是否是偏向锁	|2bit 锁标志位|	
|偏向锁	|线程ID\Epoch	|对象分代年龄	|1|01|
|轻量级锁	|指向栈中锁记录的指针	|对象分代年龄	|0|00|
|重量级锁	|指向互斥量（重量级锁）的指针	|对象分代年龄	|0|10|
|GC标记	|空	|空	|空|11|

其中轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的，稍后我们会简要分析。这里我们主要分析一下重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）

```C++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL; // 持有该对象的线程
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表(每个等待锁的线程都会被封装成ObjectWaiter对象)，
_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，
当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，
若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒。
若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。
![monitor](https://www.artima.com/insidejvm/ed2/images/fig20-1.gif)
> monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因。

### 同步块基本原理
```java
public class SyncCodeBlock {

   public int i;

   public void syncTask(){
       //同步代码库
       synchronized (this){
           i++;
       }
   }
}
```
使用 `javap` 反编译后的字节码如下：
```java
javap -c -v SyncCodeBlock
Classfile /Users/user/Downloads/SyncCodeBlock.class
  Last modified Nov 24, 2019; size 402 bytes
  MD5 checksum f5b9dd1901880f0588e13792bb15eade
  Compiled from "SyncCodeBlock.java"
public class SyncCodeBlock
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   //常量池忽略
{
  public int i;
    descriptor: I
    flags: ACC_PUBLIC

  public SyncCodeBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_0
         5: dup
         6: getfield      #2                  // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field i:I
        14: aload_1
        15: monitorexit
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             4    16    19   any
            19    22    19   any
      LineNumberTable:
        line 7: 0
        line 8: 4
        line 9: 14
        line 10: 24
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class SyncCodeBlock, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "SyncCodeBlock.java"
```
从字节码中可知同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 
指令，其中 `monitorenter` 指令指向同步代码块的开始位置， `monitorexit` 指令则指明同步代码块的结束位置，当执行 `monitorenter` 指令时，当前线程将试图获取 
objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。

如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor ，重入时计数器的值也会加 1。

倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即 `monitorexit` 指令被执行，
执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。

值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。
为了保证在方法异常完成时 `monitorenter` 和 `monitorexit` 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。
从字节码中也可以看出多了一个 `monitorexit` 指令，它就是异常结束时被执行的释放monitor 的指令。
### 同步方法基本原理
方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。
```java
public class SyncMethod {

   public int i;

   public synchronized void syncTask(){
           i++;
   }
}
```
使用 `javap` 反编译后的字节码如下：

```java
➜  javap -c -v SyncMethod.class
Classfile /Users/user/Downloads/SyncMethod.class
  Last modified Nov 24, 2019; size 284 bytes
  MD5 checksum efe2cc530afa979c7bff33ec9040de75
  Compiled from "SyncMethod.java"
public class SyncMethod
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   //常量池忽略
{
  public int i;
    descriptor: I
    flags: ACC_PUBLIC

  public SyncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public synchronized void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED // 注意这里的标记字段
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 6: 0
        line 7: 10
}
SourceFile: "SyncMethod.java"
```
从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取而代之的是ACC_SYNCHRONIZED标识，
该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

### 区别
synchronized同步代码块的时候通过加入字节码monitorenter和monitorexit指令来实现monitor的获取和释放，
也就是需要JVM通过字节码显式的去获取和释放monitor实现同步，而synchronized同步方法的时候，没有使用这两个指令，
而是检查方法的ACC_SYNCHRONIZED标志是否被设置，如果设置了则线程需要先去获取monitor，执行完毕了线程再释放monitor，也就是不需要JVM去显式的实现。

这两个同步方式实际都是通过获取monitor和释放monitor来实现同步的，而monitor的实现依赖于底层操作系统的mutex互斥原语，
而操作系统实现线程之间的切换的时候需要从用户态转到内核态，这个转成过程开销比较大。

线程获取、释放monitor的过程如下：
![](https://user-gold-cdn.xitu.io/2019/3/22/169a4d626f362741?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

线程尝试获取monitor的所有权，如果获取失败说明monitor被其他线程占用，则将线程加入到的同步队列中，等待其他线程释放monitor，
当其他线程释放monitor后，有可能刚好有线程来获取monitor的所有权，那么系统会将monitor的所有权给这个线程，
而不会去唤醒同步队列的第一个节点去获取，所以synchronized是非公平锁。如果线程获取monitor成功则进入到monitor中，并且将其进入数+1。



### Java虚拟机对synchronized的优化
在Java早期版本中，synchronized属于重量级锁，
效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，
这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。
庆幸的是在JDK6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，

为了减少获得锁和释放锁所带来的性能消耗，JDK6引入了*轻量级锁*和*偏向锁*，接下来我们简单了解一下Java官方在JVM层面对synchronized锁的优化。


锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。
#### 01偏向锁
偏向锁是JDK6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，
因此为了减少同一线程获取锁(会涉及到一些CAS操作耗时)的代价而引入偏向锁。
偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时 Mark Word 
的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，
从而也就提供程序的性能。

所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。
但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，
否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为*轻量级锁*。

获取锁的过程：

- 0.当线程请求到锁对象后立即将该锁状态标记为01，即偏向模式，然后使用CAS将当前线程ID写入到锁对象的Mark 
Word中。以后该线程可以进入同步块，连CAS都不需要，只需要检查线程ID是否匹配即可。
- 1.先检查Mark Word是否为可偏向锁的状态（即是否偏向锁即为01即表示支持可偏向锁，否则为00表示不支持可偏向锁）。
- 2.如果是支持可偏向锁，则检查Mark Word储存的线程ID是否为当前线程ID，如果是则执行同步块，否则执行步骤3。
- 3.如果检查到Mark Word存储的线程ID不是本线程的ID或者为空，则通过CAS操作去修改线程ID修改成本线程的ID
，如果修改成功（只有之前的线程释放了锁才会成功）则执行同步代码块，否则执行步骤4。
- 4.当拥有该锁的线程到达安全点Safe Point之后，挂起这个没有获取到锁的线程，升级为轻量级锁，即一旦有多线程竞争锁则升级为轻量级锁。

锁释放的过程：

- 1.有其他线程来获取这个锁，偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。
- 2.等待全局安全点。
- 3.暂停拥有偏向锁的线程，检查持有偏向锁的线程是否活着，如果不处于活动状态，则将对象头设置为*无锁状态*，否则设置为*被锁定状态*。
如果锁对象处于*无锁状态*，则恢复到无锁状态(01)，以允许其他线程竞争，如果锁对象处于*锁定状态*，则挂起持有偏向锁的线程，并将对象头Mark 
Word的锁记录指针改成当前线程的锁记录，锁升级为轻量级锁状态(00)。

关于偏向锁是怎么实现的，知乎有一个[帖子](https://www.zhihu.com/question/55075763)，可以看看。
借用于一张其中[方杰的回复](https://www.zhihu.com/question/55075763/answer/142572386)比较形象的图：
![](https://pic4.zhimg.com/80/v2-91e4ad5099644727557c5db92d2a758f_hd.jpg)
#### 00轻量级锁
倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段，此时 Mark Word 
的结构也变为轻量级锁的结构。
轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。
需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁*膨胀*为重量级锁。

与重量级锁的区别

- 重量级锁是一种悲观锁，它认为总是有多条线程要竞争锁，所以它每次处理共享数据时，不管当前系统中是否真的有线程在竞争锁，它都会使用互斥同步来保证线程的安全；

- 而轻量级锁是一种乐观锁，它认为锁存在竞争的概率比较小，所以它不使用互斥同步，而是使用CAS操作来获得锁，这样能减少互斥同步所使用的『互斥量』带来的性能开销。

借用于一张其中[方杰的回复](https://www.zhihu.com/question/55075763/answer/142572386)比较形象的图：
![](https://pic3.zhimg.com/80/v2-93c87ddbd97cb541c15c019330097733_hd.jpg)
#### 自旋锁
轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，
线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，
这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，
因此虚拟机会让当前想要获取锁的线程做几个空循环，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，
如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面*挂起*，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。
最后没办法也就只能升级为重量级锁了。

另外还有一种叫做「自适应性自旋」：
> 自适应性自旋是自旋的升级、优化，自旋的时间不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态决定。
  例如线程如果自旋成功了，那么下次自旋的次数会增多，因为JVM认为既然上次成功了，那么这次自旋也很有可能成功，那么它会允许自旋的次数更多。
  反之，如果对于某个锁，自旋很少成功，那么在以后获取这个锁的时候，自旋的次数会变少甚至忽略，避免浪费处理器资源。
  有了自适应性自旋，随着程序运行和性能监控信息的不断完善，JVM对程序锁的状况预测就会变得越来越准确，JVM也就变得越来越聪明。


#### 消除锁
消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，
通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间。
举个例子，比如 `StringBuffer` 的append是一个同步方法，但是如果在一个方法中局部变量 `StringBuffer` ，并且不会被其他线程所使用，
因此 `StringBuffer` 不可能存在共享资源竞争的情景，JVM会自动将其锁消除。
### 常见问题
#### 继承关系中的可重入性
当子类继承父类时，子类也是可以通过可重入锁调用父类的同步方法。注意由于synchronized是基于monitor实现的，因此每次重入，
monitor中的计数器仍会加1。
#### 中断与synchronized
> 当一个线程处于被阻塞状态或者试图执行一个阻塞操作时，使用Thread#interrupt()
方式中断该线程，注意此时将会抛出一个InterruptedException的异常，同时中断状态将会被复位(由中断状态改为非中断状态)，

对于中断的处理一般包括以下两种情况：
- 一种是当线程处于阻塞状态或者试图执行一个阻塞操作时（如sleep()），可以使用Thread#interrupt()对该线程进行中断，
执行中断操作后将会抛出 InterruptedException 异常并将中断状态复位。
- 另外一种是当线程处于「运行状态且不会主动捕获异常时」时，由于无法通过运行中的代码感知中断状态，需要自行主动去判断中断状态，
并编写中断线程的代码。

```java
public void run(){
    try {
        // 主动判断当前线程是否已中断
        while (!Thread.interrupted()) { // 该静态方法会对中断状态进行reset
            TimeUnit.SECONDS.sleep(2);
        }
    } catch (InterruptedException e) {
        // 被动感知被中断（通过可中断方法进行感知）
    }
}
```
需要注意的是，线程的中断操作对于正在等待获取的锁对象的synchronized方法或者代码块并不起作用，
如果一个线程在等待获取synchronized锁，要么它获得这把锁继续执行，要么它就保持等待，即使调用中断线程的方法，也不会生效。
#### 唤醒线程与synchronized的关系
> 为什么Object类的notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用？

这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，
也就是说notify/notifyAll和wait方法依赖于monitor对象，因为 monitor 存在于对象头的Mark Word 中(存储monitor引用指针)
，而synchronized关键字可以获取 monitor 。如果不使用该关键字，否则就会抛出IllegalMonitorStateException异常。

#### wait方法和锁区间
与sleep方法不同的是wait方法调用完成后，线程将被暂停，但wait方法将会释放当前持有的监视器锁(monitor)，
直到有线程调用notify/notifyAll方法后方能继续执行，而sleep方法只让线程休眠并不释放锁。同时notify/notifyAll方法调用后，
并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁，即离开锁区间时才会真正释放锁。

#### wait和notify



|Method                                 | Description                                            |
| ------------------------------------- | ------------------------------------------------------------ |
| `void wait();`                        | Enter a monitor's wait set until notified by another thread  |
| `void wait(long timeout);`            | Enter a monitor's wait set until notified by another thread or `timeout` milliseconds elapses |
| `void wait(long timeout, int nanos);` | Enter a monitor's wait set until notified by another thread or `timeout` milliseconds plus `nanos` nanoseconds elapses |
| `void notify();`                      | Wake up one thread waiting in the monitor's wait set. (If no threads are waiting, do nothing.) |
| `void notifyAll();`                   | Wake up all threads waiting in the monitor's wait set. (If no threads are waiting, do nothing.) |

### References
- https://blog.csdn.net/m_xiaoer/article/details/73274642
- https://www.artima.com/insidejvm/ed2/threadsynch.html
> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
