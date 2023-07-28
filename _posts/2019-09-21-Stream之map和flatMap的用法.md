---
layout:     post
title:      "Stream中map和flatMap差异分析"
subtitle:   "map versus flatMap in Stream"
date:       2019-09-21
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Java
    - Streaming
---
   
> Java8的Stream中引入了对流的各种计算方法，其中map和flatMap的区别常常把人搞的晕头转向，本文对此进行分析。

### 定义的区别
```java
    <R> Stream<R> map(Function<? super T, ? extends R> mapper);
    <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
```
#### 相同点
- 参数类型相同：都是一个`Function`，使用该参数对流中的元素进行转换
- 返回值类型相同：都是一个流对象`Stream<R>`

#### 不同点
- `Function`中的转换结果不同：`map`为返回流中的元素`R`，`flatMap`为一个继承了`Stream<? extends R>`类型，即它是一个`Stream`
- 也就是说，`map`中`Function`返回的`R`会原封不动的放入流中，不管它是什么类型；而`flatMap`中`Function`返回的`Stream<R>`会被拆开放入到结果流中
### 例子
OK，我们来看一个例子，假设我们有一个字符串数组：
```java
String[] strings = {"Hello World", "你 好 世 界"};
```
如何得到一个Stream对象它的每一个元素是一个单词？
#### 非扁平化
使用`map`：
```java
Stream<String[]> stream = Arrays.asList(strings).stream().
                map(str -> str.split(""));
```
这里 `map` 返回的流是 `Stream<String[]>` 类型的，因为传递给 `map` 方法的 `lambda` 为每个字符串返回了一个 `String[]` 。
继续转换：
```java
Stream<Stream<String>> streamStream = stream.map(stringArray -> Arrays.stream(stringArray));
```
这里 `map` 又将 `String[]` 转化为了 `Stream<String>` 最后得到的是 `Stream<Stream<String>>` 类型流。

通过上面两个例子可以看出， `map` 的返回结果流的类型就是直接将转化结果放了进去，如果转换结果是集合或者数组，那么它不会帮你拆解，或者说`扁平化`。
#### 扁平化
使用`flatMap`：
```java
Stream<String[]> stream1 = Arrays.asList(strings).stream().
                map(str -> str.split(""));

Stream<String> stringStream = stream1.flatMap(stringArray -> Arrays.stream(stringArray));
```
可见， `flapMap` 的转换函数虽然也是生成了一个 `Stream<String>` ，但是它内部会将这些流进行 `扁平化` ，最终返回的是 `Stream<String>` ，
而不是 `Stream<Stream<String>>` 。

对什么数据进行了 `扁平化` ？是对Stream内部元素为「数组」的Stream流对象。

`扁平化` 的结果是什么？得到一个新的Stream流对象，它的内部元素为原流对象中元素（数组）中的子元素。

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
