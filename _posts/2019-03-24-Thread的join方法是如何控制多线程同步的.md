---
layout:     post
title:      "Thread的join()是如何控制多线程同步的"
subtitle:   "join() in Class Thread with monitor lock"
date:       2019-03-24
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
---
> Thread的join()方法的使用一直很让人感到别扭，如果调用t1.join()
这个方法该怎么理解呢？其实可以理解为「加塞」，让t1加塞先执行，当前线程阻塞等待，一直到t1执行完毕再继续执行。但是join()
底层是如何让当前线程阻塞并在执行完毕后让当前线程继续运行的呢？猜测一定是依赖于t1对象的对象锁，否则无法在不增加额外锁对象同步控制的前提下使得所有调用t1.join()
方法的线程都老老实实的听从指挥。本文将通过两个例子把join机制和锁同步的关系进行说明。
    
### 阻塞与等待的区别
- 阻塞：当一个线程试图获取对象锁（非java.util
.concurrent库中的锁，即使用synchronized关键字），而该锁被其他线程持有时，则该线程进入「阻塞状态」。

它的特点是使用简单，由JVM调度器来决定唤醒自己（阻塞的线程），而不需要由另一个线程来显式唤醒自己，故不响应中断，即synchronized不支持被中断。
- 等待：当一个线程等待另一个线程通知调度器一个条件时，该线程进入「等待状态」。

它的特点是需要等待另一个线程显式地唤醒自己，实现灵活，语义更丰富，可响应中断。例如调用：Object.wait()
、Thread.join()以及等待Lock或Condition。
> 需要强调的是虽然synchronized和JUC里的Lock都实现锁的功能，但线程进入的状态是不一样的。synchronized会让线程进入阻塞态，而JUC里的Lock是用LockSupport.park()/unpark()来实现阻塞/唤醒的，会让线程进入等待态。但话又说回来，虽然等锁时进入的状态不一样，但被唤醒后又都进入runnable态，从行为效果来看又是一样的。

### Thread类的join()方法
join()方法是Thread中的一个public方法，它有几个重载版本：
```
1 join()
2 join(long millis)     //参数为毫秒
3 join(long millis,int nanoseconds)    //第一参数为毫秒，第二个参数为纳秒
```
join()实际是利用了wait()，只不过它不用等待notify()/notifyAll()，且不受其影响。它结束的条件是：1）等待时间到；2）目标线程已经run完（通过isAlive()来判断）。

先来看一下JDK源码：
```java
/**
     * Waits at most {@code millis} milliseconds for this thread to
     * die. A timeout of {@code 0} means to wait forever.
     *
     * <p> This implementation uses a loop of {@code this.wait} calls
     * conditioned on {@code this.isAlive}. As a thread terminates the
     * {@code this.notifyAll} method is invoked. It is recommended that
     * applications not use {@code wait}, {@code notify}, or
     * {@code notifyAll} on {@code Thread} instances.
     *
     * @param  millis
     *         the time to wait in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        // 0则需要一直等到目标线程run完
        if (millis == 0) {
            while (isAlive()) {// 如果被调用join方法的线程是alive状态，则调用join的方法
                wait(0); // == this.wait(0),注意这里释放的是「被调用」join方法的线程对象的锁
            }
        } else {
            // 如果目标线程未run完且阻塞时间未到，那么调用线程会一直等待。
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                /**
                * 每次最多等待delay毫秒时间后继续争抢对象锁，获取锁后继续从这里开始的下一行执行，也可能提前被notify()
                * /notifyAll()唤醒，造成delay未一次性消耗完，会继续执行while继续wait(剩下的delay)
                */
                wait(delay);
                now = System.currentTimeMillis() - base; // 这个变量now起的不太好，叫elapsedMillis就容易理解了
            }
        }
    }
```

这个方法有一些需要注意的地方：
- 它是Thread类中的方法，并不是Object类中的，所以只能让一个线程对象执行`join()`
- 它是`synchronized`修饰的方法，即执行该方法之前是需要获取该线程对象的锁的，可能会由于其他线程持有该线程对象的锁而阻塞
- 它会抛出`InterruptedException`异常，即执行过程中如果被调用的线程被中断，上层代码可以通过该方法进行捕获，并恢复运行状态
- 它只会导致调用该方法的线程释放「被调用线程的对象的this锁」，除此之外不会释放任何其他锁，比如如果调用代码在一个`synchronized`块里的话，而锁对象是一个新的对象


### 例子
关于第三点，通过一个具体的例子来模拟一下：
```java
package com.java.test;

/**
 * 模拟Thread.join()在有锁情况下是否释放锁的例子
 */
public class HelloJava {

    public static void main(String[] args) {
        String oo = new String();
        MyThread t1 = new MyThread("线程t1--", oo, null);
        MyThread t2 = new MyThread("线程t2--", oo, t1);
        t2.start(); // 注意是t2先启动
        t1.start(); // 注意t1一定要处于启动状态，否则t1.join()方法会直接返回
        try {
            t1.join(); // 等待t1结束后main才会继续执行，进入t1的wait set，t1执行完毕后会主动唤醒它
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("end");
    }

}

class MyThread extends Thread{
    private String name;
    private Object oo;
    private Thread t;
    public MyThread(String name,Object oo, Thread t){
        super(name);
        this.name = name;
        this.t = t;
        this.oo = oo;
    }
    @Override
    public void run() {
        synchronized (oo) {
            System.out.println("running in " + Thread.currentThread().getName());
            if (t != null) {
                try {
                    t.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            for(int i = 0; i < 10; i++){
                System.out.println(name + i);
            }
        }
    }
}

Output：
running in 线程t2--
(hanging)
```
程序一直处于假死状态，hang住了，用`jstack`命令看一下进程当前的stack信息。
main线程：
```
"main" #1 prio=5 os_prio=31 tid=0x00007fe965804800 nid=0x2803 in Object.wait() [0x000070000146f000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x000000076b0c86a8> (a com.java.test.MyThread)
	at java.lang.Thread.join(Thread.java:1252)
	- locked <0x000000076b0c86a8> (a com.java.test.MyThread)
	at java.lang.Thread.join(Thread.java:1326)
	at com.java.test.HelloJava.main(HelloJava.java:15)

```
首先可以看到主线程的状态是`WAITING`的，
然后下面是堆栈信息，需要从下往上看，main线程在15行处调用t1.join()，因为join方法是同步的，所以先获取了t1的对象锁，也就是
`locked <0x000000076b0c86a8> (a com.java.test.MyThread)`部分，最后在`at java.lang.Thread.join(Thread
.java:1252)`处执行了t1.wait(0)方法，导致主线程又释放了t1对象锁，所以最后就变成了`waiting on <0x000000076b0c86a8> (a com.java
.test
.MyThread)`。

t2线程：
```
"Thread-1" #12 prio=5 os_prio=31 tid=0x00007fe968812800 nid=0xa903 in Object.wait() [0x00007000028ab000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x000000076b0c86a8> (a com.java.test.MyThread)
	at java.lang.Thread.join(Thread.java:1252)
	- locked <0x000000076b0c86a8> (a com.java.test.MyThread)
	at java.lang.Thread.join(Thread.java:1326)
	at com.java.test.MyThread.run(HelloJava.java:42)
	- locked <0x000000076b0c8678> (a java.lang.String)
```
同样t2线程的状态是`WAITING`的，从下往上分析堆栈，t2先在42行代码处获取了`locked <0x000000076b0c8678> (a java.lang.String)
`锁，然后它持有了t1的对象锁`locked <0x000000076b0c86a8> (a com.java.test.MyThread)
`，进入t1.join()同步方法后，在`at java.lang.Thread.join(Thread.java:1252)`处执行了t1.wait(0)
方法，导致又释放了t1对象锁，所以最后就变成了`waiting on <0x000000076b0c86a8> (a com.java.test
.MyThread)`，t2和main线程一样，最后都在等待获取t1的对象锁。

t1对应的线程：
```
"Thread-0" #11 prio=5 os_prio=31 tid=0x00007fe968849800 nid=0xa803 waiting for monitor entry [0x00007000029ae000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.java.test.MyThread.run(HelloJava.java:39)
	- waiting to lock <0x000000076b0c8678> (a java.lang.String)
```
t1的线程状态是`BLOCKED`，被阻塞了，我们知道被阻塞一般都是没有获取到锁，这里是`waiting to lock <0x000000076b0c8678> (a java
.lang.String)`，因为该String对象锁此时被t2持有，并且t2进入了WAITING状态只能等待被唤醒。注意：`wait()`方法
虽然会释放锁，但是释放的是调用对象`obj.wait()`的对象obj锁，在这里即是释放的t1对象的锁，而不是String对象的锁。

> 关于`join()`中的`wait(0)`这里再解释一下，很多人都搞不清楚到底是什么锁，以及谁释放了这个锁。很简单wait(0)是在Thread类中的，所以是Thread对象的锁，其次t1.join
()也就意味着Thread对象是t1，但注意这里仅仅是t1的对象锁，跟t1执行线程没关系。最后，是谁释放了t1上的锁呢？当然是执行代码的线程了，那么是谁执行的t1.join()呢，很明显是
main线程和t2线程，所以这两个线程会处于`WAITING`状态，进入了t1对象锁的`wait set`集合。

### 思考题
再给一个例子，让大家自己分析。这个例子的不同在于无论是`join()`还是外层的`synchronized`都是用的一个对象锁——线程对象的锁。
```java
class A extends Thread {
   static A a;
  
   public void run() {
      try {
         synchronized(a) {
            System.out.println(Thread.currentThread().getName()+" acquired a lock on a");
            Thread.sleep(1000);
         }
      } catch (InterruptedException e){
      }
   }
  
   public static void main(String[] ar) throws Exception {
      a=new A();
      a.start();
      synchronized(a) {
         System.out.println(Thread.currentThread().getName()+" acquired a lock on a");
         a.join();
         System.out.println(Thread.holdsLock(a));
      }
   }
}
  
Output:
  
main acquired a lock on a
Thread-1 acquired a lock on a
true
```

### References
- https://coderanch.com/t/242419/certification/invocation-join-release-locks-objects
- https://segmentfault.com/q/1010000007260477

> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
