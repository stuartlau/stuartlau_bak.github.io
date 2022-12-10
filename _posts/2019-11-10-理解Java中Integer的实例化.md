---
layout:     post
title:      "理解Java中的Integer的实例化"
subtitle:   "Initialization Integer in Java"
date:       2019-11-10
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
---
> Integer类在Java中表示整数类，它是基本类型int的自动装箱的类，本文从自动装箱和内存占用两个方面分析它的实例化方法的异同。

### 如何实例化
> Integer i = 1;

上面这行代码是如何实现的呢？通过 `javap -c` 我们可以看到对应的字节码，通过分析可知，其实内部调用了 `Integer.valueOf()` 方法。

我们平时应该都用过或者见过 `parseInt` 和 `valueOf` 这两个方法。一般我们是想把 `String` 
类型的字符串数字转成 `int` 类型。从这个功能层面来说，这两个方法都一样，都可以胜任这个功能。

但是在性能和实现细节上二者有非常大的区别，想写出更高效的代码还需要对此有深刻的认识。
#### parseInt方法
`JDK` 中的实现代码如下：
```java
ublic static int parseInt(String s, int radix) throws NumberFormatException{
    /*
     * WARNING: This method may be invoked early during VM initialization
     * before IntegerCache is initialized. Care must be taken to not use
     * the valueOf method.
     */
    if (s == null) {
        throw new NumberFormatException("null");
    }

    if (radix < Character.MIN_RADIX) {
        throw new NumberFormatException("radix " + radix +" less than Character.MIN_RADIX");
    }

    if (radix > Character.MAX_RADIX) {
            throw new NumberFormatException("radix " + radix +" greater than Character.MAX_RADIX");
    }

    int result = 0;
    boolean negative = false;
    int i = 0, len = s.length();
    int limit = -Integer.MAX_VALUE;
    int multmin;
    int digit;

    if (len > 0) {
        char firstChar = s.charAt(0);
        if (firstChar < '0') { // Possible leading "+" or "-"
            if (firstChar == '-') {
                negative = true;
                limit = Integer.MIN_VALUE;
            } else if (firstChar != '+')
                 throw NumberFormatException.forInputString(s);

            if (len == 1) // Cannot have lone "+" or "-"
                 throw NumberFormatException.forInputString(s);
            i++;
        }
        multmin = limit / radix;
        while (i < len) {
            // Accumulating negatively avoids surprises near MAX_VALUE
            digit = Character.digit(s.charAt(i++),radix); //传入一个字符参数和基数，返回一个int值
            if (digit < 0) {
                throw NumberFormatException.forInputString(s);
            }
            if (result < multmin) {
                throw NumberFormatException.forInputString(s);
            }
            result *= radix;
            if (result < limit + digit) {
                throw NumberFormatException.forInputString(s);
            }
            result -= digit;
        }
    } else {
         throw NumberFormatException.forInputString(s);
    }
    return negative ? result : -result;
}
```
值得注意的是为了防止越界，默认使用了负数进行计算。

#### valueOf方法
- 字符串参数
```java
public static Integer valueOf(String s) throws NumberFormatException {
    return Integer.valueOf(parseInt(s, 10));
} 
```
可以看到，它的内部调用了 `parseInt` 方法完成字符串和 `int` 的转化，并通过重载方法 `valueOf(int) ` 完成最终到 `Integer` 的转换。

所以为什么在写代码的时候使用下列方式对基本类型数据进行赋值会被某些代码扫描工具（如 `FindBug` ）提示 `Bad Smell` 了：
> int i = Integer.valueOf("1");

因为使用 `parseInt` 方法可以更高效的完成同样的功能，因为它不需要额外的 `auto-boxing` 操作。

*注：可以通过分别对两个方法的实例化时间进行压测得出结论。此处不再粘贴实验结果*
- int参数
这个方法也是进行 `auto-boxing` 调用的方法。
```java
public static Integer valueOf(int i) {
    assert IntegerCache.high >= 127;
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
可以看到这里会对[-128, 127] 之间的数字进行特殊处理：从缓存中拿数据，也就是说不会对这些数据进行冗余保存，但前提是通过 `valueOf` 方法获取 `Integer` 对象。
如果使用构造函数，则仍然会每次实例化一个新的对象。
### IntegerCache
#### 为什么要缓存
为什么需要缓存？因为数字在程序里出现的频率实在是太高了，并且也太容易被我们忽视它们对内存的影响了。

试想一下，一个数字敏感的业务，如电商中的商品，其中包括价格属性，如果使用 `JVM` 内存来做缓存，如果商品数量超过一定程度，那么重复的价格的商品个数比例应该会很高。
此时如果每次都用 `new Integer(price)` 来存储，那么无疑对内存是一种巨大的浪费。 

#### 实现代码
```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);
            // Maximum array size is Integer.MAX_VALUE
            h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
    }

    private IntegerCache() {}
}
```
注意到它是一个内部静态类，外部无法访问，并且缓存的初始化是在 `Integer` 类加载的时候就被实例化的。

> Byte/Short/Long的缓存池范围默认都是[-128,127]。可以看出，Byte的所有值都在缓存区中，用它生成的相同值对象都是相等的。

只有 `IntegerCache` 可以通过启动参数来调整缓存的上限，`java.lang.Integer.IntegerCache.high=n` 。

### References



> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
