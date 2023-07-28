---
layout:     post
title:      "ThreadLocal原理分析和注意事项"
subtitle:   "Deep Dive into ThreadLocal"
date:       2019-02-27
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - ThreadLocal
---
    
> 本文分析`Thread`和`ThreadLocal`的关系以及实现原理和常见的问题，并提供使用`ThreadLocal`的最佳方式。

## ThreadLocal基础
先看一下 `ThreadLocal` 部分源码。
### setInitial
```java
private T setInitialValue() {
        T value = initialValue();//获取初始值
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
}
```
`initialValue`方法，默认是`null`，访问权限是`protected`，即允许重写。
```java
protected T initialValue() {
    return null;
}
```

### set 
`set` 方法实现了为线程绑定变量的工作：
```java
public void set(T value) {
    Thread t = Thread.currentThread();//1.首先获取当前线程对象
        ThreadLocalMap map = getMap(t);//2.获取该线程对象的ThreadLocalMap
        if (map != null)
            map.set(this, value); // 如果map不为空，执行set操作，以当前threadLocal对象为key，实际存储对象为value进行set操作
        else
            createMap(t, value); // 如果map为空，则为该线程创建ThreadLocalMap
    }
```
可以看到，`ThreadLocal` 不过是个「入口」，真正的变量是绑定在线程上的 `ThreadLocalMap` 变量上的。

### getMap
`getMap` 方法用来获取线程的 `ThreadLocalMap` 变量即直接访问线程的内部全局变量 `threadLocals` ：

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals; // 线程对象持有ThreadLocalMap的引用
}
```
我们来看一下这个`threadLocals` 的定义，它的类是维护在 `ThreadLocal` 类中的，访问权限为`package private`：
```java
    /* ThreadLocal values pertaining to this thread. This map is maintained by the ThreadLocal 
    class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
`ThreadLocalMap` 保存了它所属于的线程的一些数据，什么数据呢，当前线程执行过的 `ThreadLocal` 对象的初始化方法和设置值的方法。
举个例子，如有8个ThreadLocal对象，存储类型分别为Java的8
种基本类型的包装类，一个线程的初始化`ThreadLocal`操作中都会被该线程生成一个默认的值为0的对应类型的数据，所以该线程的`threadLocals`变量中存储了这8个`ThreadLocal`
对象和该线程存储的*value*，我们称之为`local value`。

### ThreadLocalMap
`ThreadLocalMap` 的 `Java Doc` 如下：

> ThreadLocalMap is a customized hash map suitable only for maintaining thread local values.
> 
> No operations are exported maintaining thread local values. No operations are exported
>
> outside of the ThreadLocal class. The class is package private to allow declaration of
> 
> fields in class Thread.  To help deal with very large and long-lived usages, the hash
> 
> table entries use WeakReferences for keys. However, since reference queues are not
>
> used, stale entries are guaranteed to be removed only when the table starts running
> 
> out of space.

可知它本质上是一个定制化的哈希表，但是并没有对外暴露任何维护哈希表的方法，如 `remove` 操作。

为了处理大量的、长期存活的数据，默认这个哈希表的 `entries`使用 `WeakReference` 作为 `key` 的类型，但是由于没有使用 `ReferenceQueue` 参数初始化 
`key` ，所以即使 `JVM` 因为内存吃紧回收了对应的 `key`的引用的 `ThreadLocal` 对象我们也无法得知并做一些清除工作，这里就埋下一个隐患给我们。

先来看看它的属性：
#### 属性
```java
    static class ThreadLocalMap {
        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0        
        ......
    }
```
可知它其实并不是一个 `Map` 接口的实现类，而是通过 `Entry` 来实现的 `key` 和 `value` 的映射关系，并用数组进行保存多个 `Entry` 。
#### 构造函数
`ThreadLocalMap`对应的构造函数如下：
```java
    /**
     * Construct a new map initially containing (firstKey, firstValue).
     * ThreadLocalMaps are constructed lazily, so we only create
     * one when we have at least one entry to put in it.
     */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
```
可知每个 `Entry` 实例持有了 `key` 为`ThreadLocal`对象实例，`value` 为具体的` ThreadLocal<T>` 泛型 `T` 的实例对象。初始化的数组大小为 
`16` 。
#### Entry
> The entries in this hash map extend WeakReference, using its main ref field as the key (which
> 
> is always a ThreadLocal object). Note that null keys (i.e. entry.get() == null) mean that
> 
> the key is no longer referenced, so the entry can be expunged from table.  Such entries
> 
> are referred to as "stale entries" in the code that follows.
       
```java
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```
注意到 `Entry` 继承了`WeakReference` 类，而弱引用本身在 `GC` 触发时会被回收，所以 `key` 
可能会变为`null`，即对应的引用对象被回收掉了，但是 `value` 是一个「强引用」。该 `Entry` 在table
数组中也不会被垃圾回收自动触发「缩容」被删除掉，不过`ThreadLocalMap`为我们提供了很多`expunge
`机制来清理对应的过期数据，但这个机制「需要显式触发」，这里也就是可能出现内存泄漏的地方！

#### expungeStaleEntry
来看一下`expungeStaleEntry` 方法：
```java
    /**
     * Expunge a stale entry by rehashing any possibly colliding entries
     * 
     * lying between staleSlot and the next null slot.  This also expunges
     * 
     * any other stale entries encountered before the trailing null.  See
     * 
     * Knuth, Section 6.4
     *
     * @param staleSlot index of slot known to have null key
     * 
     * @return the index of the next null slot after staleSlot
     * 
     * (all between staleSlot and this slot will have been checked
     * 
     * for expunging).
     */
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // expunge entry at staleSlot
        tab[staleSlot].value = null; // 清除对应Entry的value的值，即释放对value的强引用
        tab[staleSlot] = null; // 清除ThreadLocalMap中的Entry[]中对应i位置的Entry，回收内存
        size--; // 数组缩容

        // Rehash until we encounter null
        Entry e;
        int i;
        for (i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == null) { // 如果k已经被回收，即ThreadLocal对象被回收
                e.value = null; // 清除对应Entry的value的值，即释放对value的强引用
                tab[i] = null; // 清除ThreadLocalMap的的Entry[]中对应i位置的Entry，回收内存
                size--; // 数组缩容
            } else {
                int h = k.threadLocalHashCode & (len - 1);
                if (h != i) {
                    tab[i] = null;

                    // Unlike Knuth 6.4 Algorithm R, we must scan until
                    // null because multiple entries could have been stale.
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        return i;
    }
    
    /**
     * Expunge all stale entries in the table.
     */
    private void expungeStaleEntries() {
        Entry[] tab = table;
        int len = tab.length;
        for (int j = 0; j < len; j++) {
            Entry e = tab[j];
            if (e != null && e.get() == null) // 清理Entry不存在或对应的key已经过期的数据
                expungeStaleEntry(j);
        }
    }
```
很幸运的是，这个方法在内部很多方法实现里都会被调用到，比如：`ThreadLocal#remove()`， `ThreadLocal#set()`。

所以当我们显式调用这些方法的时候内部会自动帮我们清理那些已经被垃圾回收的 `key` 对应的 `Entry` 以及它引用的 `value` 的强引用。

#### set
来看一下 `ThreadLocalMap`的 `set()` 的实现：
```java
    /**
     * Set the value associated with key.
     *
     * @param key the thread local object
     * @param value the value to be set
     */
    private void set(ThreadLocal<?> key, Object value) {

        // We don't use a fast path as with get() because it is at
        // least as common to use set() to create new entries as
        // it is to replace existing ones, in which case, a fast
        // path would fail more often than not.

        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) { // 更新value
                e.value = value;
                return;
            }

            if (k == null) { // 对应位置上的老的ThreadLocal对象已经被回收，替换为新的key
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        // 如果当前的Entry[]中不存在对应的ThreadLocal的数据，则实例化一个新的Entry并触发扩容
        // 如果超过阈值会触发rehash，内部会调用
        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
    
    /**
     * Re-pack and/or re-size the table. First scan the entire
     * table removing stale entries. If this doesn't sufficiently
     * shrink the size of the table, double the table size.
     */
    private void rehash() {
        expungeStaleEntries();

        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
            resize();
    }
```
## 线程和ThreadLocal
### 一次性线程
如果线程是通过手动`new`的`Thread`，它会在线程退出时帮我们清理线程占用的紫云，原因是这类线程的生命周期随着`run`方法的结束就结束了，`GC
`会帮我们回收该实例引用的内存空间，包括堆栈信息和`ThreadLocalMap`中的内容。

当线程退出时会执行`Thread#exit()`方法：
```java
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```
从源码可以看出当线程结束时，会令`threadLocals=null`，也就意味着`GC`的时候就可以将`threadLocalMap`进行垃圾回收，换句话说`threadLocalMap`
生命周期实际上和`thread`的生命周期相同。

### 线程池
如果当前线程是通过线程池进行维护的且属于核心线程，即一旦被启动后在不关闭线程池之前不会被回收。则需要注意该线程由于会被调用多次，则内部的`threadLocals
`中的数据如果不及时清理会被复用，导致可能的逻辑错误。

> 如果认为`ThreadLocal`是保证每个线程的数据彼此独立互不干扰，则可能会在使用时产生一定危险。
>
> 主要体现在如spring的事务管理，包括Hibernate的session管理等都有出现，在web开发中，有时会用来管理用户会话
> 
> HttpSession，web交互中这种典型的「一个请求一个线程」的场景似乎比较适合使用ThreadLocal，但是需要特别注意的是，
>
> 由于此时session与线程关联，而tomcat这些web服务器多会采用线程池机制，也就是说线程是可复用的，所以在每一次进入的时候都需要重新进行set，或者在结束时及时remove。

## 正确的使用方式
- 尽量使用`static`的全局`ThreadLocal`变量，保证各个线程的`threadLocals`中只会有一个`Entry`引用
- 不要一次性初始化大量的、局部`ThreadLocal`变量，还不如直接使用临时变量替代
- 业务（线程）在使用完毕`ThreadLocal`后要及时调用`remove()`方法及时释放`threadLocals`中的内存资源
- 或者（线程）每次使用前都调用`set()`也是一样可以达到及时清理`staleEntry`的效果的

## 为什么使用弱引用？
从文章开头通过ThreadLocal，ThreadLocalMap和Entry的引用关系看起来`ThreadLocal`存在内存泄漏的问题似乎是因为`ThreadLocal`实例是被
是被弱引用修饰的。那为什么要使用弱引用呢？

> 如果使用强引用

假设`ThreadLocal`中`Entry`里使用的是强引用到ThreadLocal实例，在业务代码中执行threadLocalInstance=null操作，以清理掉threadLocalInstance
的目的，但是因为threadLocalMap
的Entry强引用threadLocalInstance，因此在GC的时候进行可达性分析，threadLocalInstance依然可达，对threadLocalInstance
并不会进行垃圾回收，这样就无法真正达到业务逻辑的目的，出现逻辑错误。

> 如果使用弱引用

假设Entry弱引用threadLocal变量，尽管可能会出现内存泄漏的问题，但是在threadLocal的生命周期里（`set`,`getEntry`,
`remove`）里，都会针对key为`null`的「脏Entry」进行处理：清除或更新。
从以上的分析可以看出，使用弱引用的话在`ThreadLocal`生命周期里会尽可能的保证不出现内存泄漏的问题，达到安全的状态。

## References
- https://www.jianshu.com/p/dde92ec37bd1
- https://blog.csdn.net/zsfsoftware/article/details/50933151

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
