---
layout:     post
title:      "ConcurrentHashMap的弱一致性分析"
subtitle:   "ConcurrentHashMap's Weak Consistency"
date:       2019-12-08
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - ConcurrentHashMap
---
    
> ConcurrentHashMap是Java中的一个经典的「读不加锁写加锁」的数据结构，但它并不是强一致性的，本文基于JDK1.6对其特性进行一定的分析。

#### ConcurrentHashMap#get
get方法是弱一致的，是什么含义？可能你期望往ConcurrentHashMap底层数据结构中加入一个元素后，立马能对get可见，但ConcurrentHashMap并不能如你所愿。换句话说，put操作将一个元素加入到底层数据结构后，get可能在某段时间内还看不到这个元素，若不考虑内存模型，单从代码逻辑上来看，却是应该可以看得到的。

下面将结合代码和java内存模型相关内容来分析下put/get方法（本文中所有ConcurrentHashMap相关的代码均来自hotspot1.6.0_18）。put方法我们只需关注Segment#put，get方法只需关注Segment#get，在继续之前，先要说明一下Segment里有两个volatile变量：count和table；HashEntry里有一个volatile变量：value。

Segment#put

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
	lock();
	try {
		int c = count;
		if (c++ > threshold) // ensure capacity
			rehash();
		HashEntry<K,V>[] tab = table;
		int index = hash & (tab.length - 1);
		HashEntry<K,V> first = tab[index];
		HashEntry<K,V> e = first;
		while (e != null && (e.hash != hash || !key.equals(e.key)))
			e = e.next;

		V oldValue;
		if (e != null) {
			oldValue = e.value;
			if (!onlyIfAbsent)
				e.value = value;
		}
		else {
			oldValue = null;
			++modCount;
			tab[index] = new HashEntry<K,V>(key, hash, first, value);
			count = c; // write-volatile
		}
		return oldValue;
	} finally {
		unlock();
	}
}
```

Segment#get

```java
V get(Object key, int hash) {
	if (count != 0) { // read-volatile
		HashEntry<K,V> e = getFirst(hash);
		while (e != null) {
			if (e.hash == hash && key.equals(e.key)) {
				V v = e.value;
				if (v != null)
					return v;
				return readValueUnderLock(e); // recheck
			}
			e = e.next;
		}
	}
	return null;
}
```
我们如何确定线程1放入某个变量的值是否对线程2可见？文章开头提到的JLS链接中有说到，当a hb c时，a对c可见，那么我们接下来我们只要寻找put和get之间所有可能的执行轨迹上的hb关系。
要找出hb关系，我们需要先找出与hb相关的Action。为方便，这里将两段代码放到了一张图片上。
![](http://ifeve.com/wp-content/uploads/2013/04/concurrentHashMap_put_get.png)

可以注意到，同一个Segment实例中的put操作是加了锁的，而对应的get却没有。根据hb关系中的线程间Action类别，可以从上图中找出这些Action，主要是volatile读写和加解锁，也就是图中画了横线的那些。

put操作可以分为两种情况，一是key已经存在，修改对应的value；二是key不存在，将一个新的Entry加入底层数据结构。

key已经存在的情况比较简单，即if (e != null)部分，前面已经说过HashEntry的value是个volatile变量，当线程1给value赋值后，会立马对执行get的线程2可见，而不用等到put方法结束。

key不存在的情况稍微复杂一些，新加一个Entry的逻辑在else中。那么将new HashEntry赋值给tab[index]是否能立刻对执行get的线程可见呢？我们只需分析写tab[index]与读取tab[index]之间是否有hb关系即可。



| 执行put的线程                                             | 执行get的线程                  |
| --------------------------------------------------------- | ------------------------------ |
| ⑧tab[index] = new HashEntry<K,V>(key, hash, first, value) |                                |
| ②count = c                                                |                                |
|                                                           | ③if (count != 0)               |
|                                                           | ⑨HashEntry e = getFirst(hash); |

tab变量是一个普通的变量，虽然给它赋值的是volatile的table。另外，虽然引用类型（数组类型）的变量table是volatile的，但table中的元素不是volatile的，因此⑧只是一个普通的写操作；count变量是volatile的，因此②是一个volatile写；③很显然是一个volatile读；⑨中getFirst方法中读取了table，因此包含一个volatile读。

根据Synchronization Order，对同一个volatile变量，有volatile写 hb volatile读。在这个执行轨迹中，时间上②在③之前发生，且②是写count，③是读count，都是针对同一个volatile变量count，因此有② hb ③；又因为⑧和②是同一个线程中的，③和⑨是同一个线程中的，根据Program Order，有⑧ hb ②，③ hb ⑨。目前我们有了三组关系了⑧ hb ②，② hb ③，③ hb ⑨，再根据hb关系是可传递的（即若有x hb y且y hb z，可得出x hb z），可以得出⑧ hb ⑨。因此，如果按照上述执行轨迹，⑧中写入的数组元素对⑨中的读取操作是可见的。

再考虑这样一个执行轨迹：



| 执行put的线程                                             | 执行get的线程                  |
| --------------------------------------------------------- | ------------------------------ |
| ⑧tab[index] = new HashEntry<K,V>(key, hash, first, value) |                                |
|                                                           | ③if (count != 0)               |
| ②count = c                                                |                                |
|                                                           | ⑨HashEntry e = getFirst(hash); |

这里只是变换了下执行顺序。每条语句的volatile读写含义同上，但它们之间的hb关系却改变了。Program Order是我们一直拥有的，即我们有⑧ hb ②，③ hb ⑨。但这次对volatile的count的读时间上发生在对count的写之前，我们无法得出② hb ⑨这层关系了。因此，通过count变量，在这个轨迹上是无法得出⑧ hb ⑨的。那么，存不存在其它可替换关系，让我们仍能得出⑧ hb ⑨呢？

我们要找的是，在⑧之后有一条语句或指令x，在⑨之前有一条语句或指令y，存在x hb y。这样我们可以有⑧ hb x，x hb y， y hb ⑨。就让我们来找一下是否存在这样的x和y。图中的⑤、⑥、⑦、①存在volatile读写，但是它们在⑧之前，因此对确立⑧ hb ⑨这个关系没有用处；同理，④在⑨之后，我们要找的是⑨之前的，因此也对这个问题无益。前面已经分析过了②，③之间没法确立hb关系。

在⑧之后，我们发现一个unlock操作，如果能在⑨之前找到一个lock操作，那么我们要找的x就是unlock，要找的y就是lock，因为Synchronization Order中有unlock hb lock的关系。但是，很不幸运，⑨之前没有lock操作。因此，对于这样的轨迹，是没有⑧ hb ⑨关系的，也就是说，如果某个Segment实例中的put将一个Entry加入到了table中，在未执行count赋值操作之前有另一个线程执行了同一个Segment实例中的get，来获取这个刚加入的Entry中的value，那么是有可能取不到的！

此外，如果getFirst(hash)先执行，tab[index] = new HashEntry<K,V>(key, hash, first, value)后执行，那么，这个get操作也是看不到put的结果的。

#### ConcurrentHashMap#clear
clear方法很简单，看下代码即知。
```java
public void clear() {
	for (int i = 0; i < segments.length; ++i)
		segments[i].clear();
}
```
因为没有全局的锁，在清除完一个segments之后，正在清理下一个segments的时候，已经清理segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。


#### ConcurrentHashMap中的迭代器
ConcurrentHashMap中的迭代器主要包括entrySet、keySet、values方法。它们大同小异，这里选择entrySet解释。
当我们调用entrySet返回值的iterator方法时，返回的是EntryIterator，在EntryIterator上调用next方法时，最终实际调用到了HashIterator.advance()方法，看下这个方法：
```java
final void advance() {
    // 如果下一个entry存在，即nextEntry.next!=null则当前指针前进一位，方法返回
	if (nextEntry != null && (nextEntry = nextEntry.next) != null)
		return;

    // 否则说明已经到了当前segment的尽头，需要从下一个继续开始遍历
	while (nextTableIndex >= 0) {
	    // currentTable中的下一个位置继续判断，如果还有，并且头元素不为null
	    // 则当前指针前进一位，方法返回
		if ( (nextEntry = currentTable[nextTableIndex--]) != null)
			return;
	}

    // 否则说明遍历完了全部segment，没有数据了
	while (nextSegmentIndex >= 0) {
		Segment<K,V> seg = segments[nextSegmentIndex--];
		if (seg.count != 0) {
			currentTable = seg.table;
			for (int j = currentTable.length - 1; j >= 0; --j) {
				if ( (nextEntry = currentTable[j]) != null) {
					nextTableIndex = j - 1;
					return;
				}
			}
		}
	}
}
```
这个方法在遍历底层数组。在遍历过程中，如果已经遍历的数组上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。
如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中。这就是ConcurrentHashMap迭代器弱一致的表现。
#### 总结
ConcurrentHashMap的弱一致性主要是为了提升效率，是一致性与效率之间的一种权衡。要成为强一致性，就得到处使用锁，甚至是全局锁，这就与HashTable和同步的HashMap一样了。
另外一种数据结构COW也是一样的，通过弱一致性保证读不加锁写加锁，大大提供了读的并发粒度，可以参考[Java中的Copy-On-Write]({{ site.url }}/2019/11/16/Java中的Copy-0n-Write/)。


### References
- http://ifeve.com/concurrenthashmap-weakly-consistent/
- https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
