---
layout:     post
title:      "Java中的Copy-On-Write"
subtitle:   "Copy-On-Write in Java"
date:       2019-11-16
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
---
> Copy-On-Write简称COW，是一种用于程序设计中的优化策略。其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容Copy出去形成一个新的内容然后再改，这是一种延时懒惰策略。
>
> 本文从Java语言的角度谈一下常用API和应用场景。

### CopyOnWriteArrayList
顾名思义，本质上继承了List接口，底层使用数组来实现存储，并使用 `volatile` 关键字修饰数组对象，保证多线程间的数据读取都是最新的主内存的数据：
```
/** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```
#### write op
写操作是通过单独复制一份新的数组来完成新数据插入的，并通过替换原有数组的属性的指向来完成读取数据源的指向改变。
```java
/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}
```
写并不会阻塞读，但写操作在同一时刻只能有一个线程执行，另外写还有一个区别于 `ConcurrentHashMap` 这种原子写的数据结构的优势：它支持批量写，如addAll()
和removeAll()方法：
```java
public boolean addAll(Collection<? extends E> c) {
    Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
        ((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
    if (cs.length == 0)
        return false;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (len == 0 && cs.getClass() == Object[].class)
            setArray(cs);
        else {
            Object[] newElements = Arrays.copyOf(elements, len + cs.length);
            System.arraycopy(cs, 0, newElements, len, cs.length);
            setArray(newElements);
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```
排序本质上是改变数组的结构，也是写操作：
```java
public void sort(Comparator<? super E> c) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        Object[] newElements = Arrays.copyOf(elements, elements.length);
        @SuppressWarnings("unchecked") E[] es = (E[])newElements;
        Arrays.sort(es, c);
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```
#### read op
```java
public int size() {
    return getArray().length;
}

private E get(Object[] a, int index) {
    return (E) a[index];
}

public E get(int index) {
    return get(getArray(), index);
}

```
可见，读的时候不需要加锁，如果读的时候有多个线程正在向ArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的ArrayList。

#### scenario
- reads hugely outnumber writes;
- the array is small (or writes are very infrequent);
- the caller genuinely needs the functionality of a list rather than an array.

### CopyOnWriteArraySet
一个包装类，内部通过操作 `CopyOnWriteArrayList` 来实现：
```java
private final CopyOnWriteArrayList<E> al;

/**
 * Creates an empty set.
 */
public CopyOnWriteArraySet() {
    al = new CopyOnWriteArrayList<E>();
}
```
#### add
为了保证元素的唯一，使用了 `CopyOnWriteArrayList` 的 `addIfAbsent` 方法：
- 先判断当前的快照结构中是否存在，此时不加锁
- 如果存在直接返回false
- 如果不存在则通过加锁做插入，但插入前又做了一次判断，防止已经有线程先于本操作插入了新元素

```java
/**
 * Appends the element, if not present.
 */
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray(); // 获取快照
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

/**
 * static version of indexOf, to allow repeated calls without
 * needing to re-acquire array each time.
 */
// 使用静态方法来遍历，防止使用数组自身的indexOf每次都去重新获取数组引用
private static int indexOf(Object o, Object[] elements, int index, int fence) {
    if (o == null) {
        for (int i = index; i < fence; i++)
            if (elements[i] == null)
                return i;
    } else {
        for (int i = index; i < fence; i++)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}
/**
 * A version of addIfAbsent using the strong hint that given
 * recent snapshot does not contain e.
 */
private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) { // 再次检查和当前的加锁快照是否相同
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
注意它和HashSet的区别:
- 纯数组存储，插入和查找都是顺序扫描，比HashSet要快
- 如果讲一个已经在集合内的元素进行添加的话HashSet有一定的性能开销，而CopyOnWriteArraySet有专门的优化

> The second point comes from the fact that CopyOnWriteArraySet relies on the addIfAbsent() 
method provided by CopyOnWriteArrayList. This, as an atomic operation, checks whether an equal element is already in the list and adds it only if it is not. This method, and thus the set implementation, has the subtle characteristic that a copy of the array is made (and then thrown away) even if the item isn't added. Of course, that doesn't make the already-present case any more expensive than the not-present case, but it means that the already-present case isn't generally much cheaper either. (Both cases also perform similarly with HashSet, but with the latter class, both are generally much less expensive since only a tiny subset of the set is scanned when inserting or retrieving.)

### CopyOnWriteMap 
默认JDK并没有实现，不过在了解了原理之后，我们自己写一个也并不是什么看困难的事情。
下面的代码出自[ifeve](http://ifeve.com/java-copy-on-write/)，虽然有点过于简陋，不过理解一下大概意思即可。
```java
import java.util.Collection;
import java.util.Map;
import java.util.Set;

public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }

    public V put(K key, V value) {

        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }

    public V get(Object key) {
        return internalMap.get(key);
    }

    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```
### CopyOnWrite的缺点
CopyOnWrite容器有很多优点，但是同时也存在两个问题：
- 内存占用问题。

因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。
如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Young GC和Full 
GC。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
- 数据一致性问题。

CopyOnWrite容器只能保证数据的最终一致性 `Eventual 
Consistency`，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

### References
- http://ifeve.com/java-copy-on-write/

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
