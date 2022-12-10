---
layout:     post
title:      "Java8函数式编程之Reduction"
subtitle:   "Reduction in Java8"
date:       2019-09-11
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Algorithm
    - Java
    - Reduction
---
    

> Java8中引入了函数式计算以及Lambda和Stream等特性，其中的流式计算引入了收集器、组合器等规约操作用到概念，非常值得我们好好学习。
>
> 本文以「规约」为线索，先从Stream的reduce方法说起，然后延展到collect方法，以及Collector接口即Collectors中常用的规约方法。

### 什么是规约
先来看一段`Java Doc` 中对规约操作的说明：
> Reduction operations
>
>  A reduction operation (also called a fold) takes a sequence of input elements and combines
> 
> them into a single summary result by repeated application of a combining operation, such as
> 
> finding the sum or maximum of a set of numbers, or accumulating elements into a list. The
> 
> streams classes have multiple forms of general reduction operations, called reduce() and 
>
> collect(), as well as multiple specialized reduction forms such as sum(), max(), or count().
>
> Of course, such operations can be readily implemented as simple sequential loops, as in:
>       int sum = 0;
>
>       for (int x : numbers) {
>
>          sum += x;
>
>       }

可知，规约操作又称为 `fold` 折叠，它将一个序列的输入元素进行聚合后生成一个结果，操作过程是不断的重复「聚合操作」，比如比较数字大小或者加和操作。
虽然可以通过串行循环的方式执行上述计算，但是性能确实不高。
> However, there are good reasons to prefer a reduce operation over a mutative accumulation
> 
> such as the above. Not only is a reduction "more abstract" -- it operates on the stream as a
> 
> whole rather than individual elements -- but a properly constructed reduce operation is
> 
> inherently parallelizable, so long as the function(s) used to process the elements are 
>
> associative and stateless.

因为规约操作是一种抽象的操作，它将数据流看成一个整体而不是独立的一个一个元素，所以并行计算在规约操作中十分常见，前提是代操作的元素和函数是无状态的和可组合的。

> Reduction parallellizes well because the implementation can operate on subsets of the data in
> 
> parallel, and then combine the intermediate results to get the final correct answer. (Even if
> 
> the language had a "parallel for-each" construct, the mutative accumulation approach would still
>
> required the developer to provide thread-safe updates to the shared accumulating variable sum,
> 
> and the required synchronization would then likely eliminate any performance gain from
> 
> parallelism.) Using reduce() instead removes all of the burden of parallelizing the reduction
> 
> operation, and the library can provide an efficient parallel implementation with no additional
> 
> synchronization required.

而规约操作在并行执行时一般都需要考虑线程安全问题，如并发更新操作可能因为不同的调度导致结果不同，导致开发者还需要考虑各种同步的问题，这样会降低并行计算带来的好处。
但 `Java` 提供的`reduce()` 函数可以免去开发者对数据和操作进行额外的同步控制，一切都由底层自动帮我们完成，非常的方便。
 
### Stream#reduce()

`Stream` 带有一个 `reduce` 方法，通过该方法我们可以实现 `count` 、 `max` 、`min` 、 `sum` 等功能，非常强大。
规约操作可以将流的所有元素组合成一个结果，该方法有三种重载版本。

#### 1. 一个参数版本

> Optional<T> reduce(BinaryOperator<T> accumulator)

该方法无初始值，所以返回值使用 `Optional` 表示结果可能不存在。
使用 `Java` 代码来表述如下：

```java
T result = null; 
if (a == null) return result;
T identity = a[0];
result = identity;
for (int i = 1; i < n; i++) {
   result = accumulator.apply(result, a[i]);  
}
return result;  
```

这里的 `identity` 其实就是第一个元素，整体计算次数为 `n - 1` 。计算的顺序为，a[0]与a[1]进行二合运算，结果与a[2]做二合运算，一直到最后与a[n-1]做二合运算。

#### 2. 两个参数版本

> T reduce(T identity, BinaryOperator<T> accumulator)

`identity`：循环计算的初始值
`accumulator`：计算的累加器，其方法签名为`apply(T t,U u)`，在该 `reduce` 方法中第一个参数`t`为上次该函数计算的返回值，第二个参数`u`为`Stream`中的元素，这个函数把这两个值计算`apply`，得到的和会被赋值给下次执行这个方法的第一个参数。

使用`Java`代码来表述如下：

```java
T result = identity; 
for (int i = 0; i < n; i++) {
   result = accumulator.apply(result, a[i]);  
}
return result; 
```

注意区分与一个参数的 `reduce` 方法的不同：它多了一个初始化的值，因此计算的顺序是`identity` 与a[0]进行二合运算，结果与a[1]再进行二合运算...，最终与a[n-1]进行二合运算，一共计算 `n` 次。

举一个小例子：
```java
int value = Stream.of(1, 2, 3, 4).reduce(100, (sum, item) -> sum + item);
Assert.assertSame(value, 110);
/* 或者使用方法引用 */
value = Stream.of(1, 2, 3, 4).reduce(100, Integer::sum);
```

#### 3. 三个参数版本

> \<U\> U reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)

- `identity` - the identity value for the combiner function
- `accumulator` - an associative, non-interfering, stateless function for incorporating an 
additional element into a result
- `combiner` - an associative, non-interfering, stateless function for combining two values, which 
must be compatible with the accumulator function

乍一看与两个参数的 `reduce` 方法几乎一致，但是 `accumulator` 的类型变成了 `BiFunction` 而不是 `BinaryOperator` ，并且还多了一个 `combiner` 参数，而它的类型是第二个方法里 `accumulator` 参数的类型—— `BinaryOperator` 。
其实`BinaryOperator`接口是实现了`BiFunction`接口的，定义如下：

> public interface BinaryOperator<T> extends BiFunction<T, T ,T>

也就是说 `BiFunction` 的三个参数类型可以是一样的也可以完全不同，而 `BinaryOperator` 直接限定了三个参数类型必须相同。

OK，我们都知道`accumulator`是实现「累加」操作的，那么`combiner`的作用是什么呢？
如果你用 `串行流` 的方式来调用这个方法的话你会发现 `combiner` 并没有被调用，所有计算都在 `accumulator` 中执行，并返回结果。如下面的实例代码：

```java
Integer ageSum = persons
    .stream()
    .reduce(0,
        (sum, p) -> {
            System.out.format("accumulator: sum=%s; person=%s\n", sum, p);
            return sum += p.age;
        },
        (sum1, sum2) -> {
            System.out.format("combiner: sum1=%s; sum2=%s\n", sum1, sum2);
            return sum1 + sum2;
        });

// accumulator: sum=0; person=Max
// accumulator: sum=18; person=Peter
// accumulator: sum=41; person=Pamela
// accumulator: sum=64; person=David
```

其实这个组合器是在 `并行流` 的执行方式时被调用的，因为累加器会被并行调用，所以需要组合器用于计算各个子累加值的总和。

在上面的参数描述中，我们看到了一些特性：
1. `associative`，可组合的，`JavaDoc` 中的描述如下：
> Associativity
>
> An operator or function op is associative if the following holds:
>
>      (a op b) op c == a op (b op c)
>  
>  
>The importance of this to parallel evaluation can be seen if we expand this to four terms:
>
>      a op b op c op d == (a op b) op (c op d)
>  
>  
> So we can evaluate (a op b) in parallel with (c op d), and then invoke op on the results.
>
> Examples of associative operations include numeric addition, min, and max, and string concatenation.

可见我们常用的数学计算，如加减乘数，求最大最小值都符合这一类「可组合」的条件，在并行计算时这种特性可以被充分利用。

2. `Non-interference`，无干扰，`JavaDoc` 中的描述如下：

>Streams enable you to execute possibly-parallel aggregate operations over a variety of data sources,
> 
>including even non-thread-safe collections such as ArrayList. This is possible only if we can prevent
> 
>interference with the data source during the execution of a stream pipeline. 

举一个 `interference` 的例子：
> For well-behaved stream sources, the source can be modified before the terminal operation commences
> 
> and those modifications will be reflected in the covered elements. For example, consider the
> 
> following code:
>
>      List<String> l = new ArrayList(Arrays.asList("one", "two"));
>
>      Stream<String> sl = l.stream();
>
>      l.add("three");
>
>      String s = sl.collect(joining(" "));
  
> First a list is created consisting of two strings: "one"; and "two". Then a stream is created from
> 
> that list. Next the list is modified by adding a third string: "three". Finally the elements of
 the
> 
> stream are collected and joined together. Since the list was modified before the terminal collect
> 
> operation commenced the result will be a string of "one two three".
 
由于在未调用`terminal` 操作之前，是可以对`stream source` 进行操作的，如添加或删除元素，该行为会在 `terminal`操作的时候反映在结果中。

3.`Stateless`，无状态，`JavaDoc` 中的描述如下：
>  Stream pipeline results may be nondeterministic or incorrect if the behavioral parameters to the
> 
>  stream operations are stateful. A stateful lambda (or other object implementing the appropriate
> 
>  functional interface) is one whose result depends on any state which might change during the
> 
>  execution of the stream pipeline. An example of a stateful lambda is the parameter to map() in:
>
>        Set<Integer> seen = Collections.synchronizedSet(new HashSet<>());
>
>        stream.parallel().map(e -> { if (seen.add(e)) return 0; else return e; })...
>
>    
>  Here, if the mapping operation is performed in parallel, the results for the same input could vary
>
>   from run to run, due to thread scheduling differences, whereas, with a stateless lambda expression
>
>   the results would always be the same.
>
>  Note also that attempting to access mutable state from behavioral parameters presents you with a
> 
>  bad choice with respect to safety and performance; if you do not synchronize access to that state,
> 
>  you have a data race and therefore your code is broken, but if you do synchronize access to that
> 
>  state, you risk having contention undermine the parallelism you are seeking to benefit from.
> 
>  The best approach is to avoid stateful behavioral parameters to stream operations entirely;
> 
>  there is usually a way to restructure the stream pipeline to avoid statefulness.

无状态，保证了在并行执行的时候可以得到相同的结果，如果依赖中间的状态，则由于并发调度的顺序不同，每次得到的结果是不同的。


#### 特点
> `reduce()` performs an immutable reduction (i.e reduction produces a new value/object).
  
`reduce` 方法每次总是返回一个新的值，`accumulator` 也是每次处理元素的时候返回一个新值。所以如果你想将流中的元素规约成一个更复杂的对象，如集合，这样的效率就非常低了。
比如每次你都要将元素加到集合中，那么每次`accumulator` 都会生成一个新的集合对象，仅包含这次处理的元素，堆内存也造成了一定的浪费。

如果你想用更高效的方法，应该每次更新一个已有集合的状态，也就是下面引出的 `Stream.collect` 。

### Stream#collect()

虽然都是处理「规约」，但是它与`reduce` 的一个中最重要的区别就是该方法是负责处理可变式规约——`Mutable Reduction`的，`Java Doc`中该定义解释如下：

> A mutable reduction operation accumulates input elements into a mutable result container,
>
> such as a Collection or StringBuilder, as it processes the elements in the stream.
>
> If we wanted to take a stream of strings and concatenate them into a single long string, we
>
> could achieve this with ordinary reduction:
>
> ```
> String concatenated = strings.reduce("", String::concat)
> ```
>
> We would get the desired result, and it would even work in parallel. However, we might not be
>
> happy about the performance! Such an implementation would do a great deal of string copying, and
>
> the run time would be O(n^2) in the number of characters. A more performant approach would be
>
> to accumulate the results into a StringBuilder, which is a mutable container for accumulating
>
> strings. We can use the same technique to parallelize mutable reduction as we do with ordinary reduction.

对于字符串拼接的例子来说，如果用传统的 `reduce` 方式实现，每次创建一个新的字符串的方式（ `String::concat` 方法每次生成一个新的字符串），
这样非常浪费内存空间。而 `collect` 方法本质上是「变更」了「container」的状态，而不是用新的值去替换旧值，
这也就是为什么要求存储变量的容器是`a mutable result container`。

所以使用`Collection` 或 `StringBuilder` 这种可变的容器来解决这类问题效果要好很多：
```java
R result = supplier.get();
for (T element : this stream)
    accumulator.accept(result, element);
return result;
```
上述伪代码展示了这种实现的好处，只是用一个 `result` 参与计算并作为结果返回。

好了，有了这些知识我们来看一下之前的例子，对于字符串拼接来说，如果用 `collect` 方法来实现的版本为：

```java
String concat = stringStream.collect(StringBuilder::new, StringBuilder::append,
                                               StringBuilder::append).toString();
```
#### 特点
`collect` 方法执行的是「可变规约」：
> Performs a mutable reduction (i.e. mutates the resulting object). Needed to apply a reduction 
performed by a mutating method of a mutable type.

#### 1. 三个参数版本
它的三参数方法声明如下：   

> \<R\> R collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)

- `supplier` - a function that creates a new result container. For a parallel execution, this 
function may be called multiple times and must return a fresh value each time.
- `accumulator` - an associative, non-interfering, stateless function for incorporating an 
additional element into a result
- `combiner` - an associative, non-interfering, stateless function for combining two values, which 
must be compatible with the accumulator function

上述三个参数和之前的`Stream#reduce` 方法的三参数版本的说明除了 `supplier` 要求是一个新容器之外的描述是一致的，只不过类型不一样而已。


再举一个 `Collection` 例子，这个例子即为过滤大于2的元素，将剩余结果收集到一个新的list中：

```java
Stream<Integer> stream = Stream.of(1, 2, 3, 4).filter(p -> p > 2);
List<Integer> result = stream.collect(() -> new ArrayList<>(), (list, item) -> list.add(item), (one, two) -> one.addAll(two));
/* 或者使用方法引用 */
result = stream.collect(ArrayList::new, List::add, List::addAll);
```
我们来分析一下 `collect` 方法中三个参数的实例化：
- 第一个参数即`supplier`，要求每次都生成一个新的 `ArrayList` ，这里直接 `new` 了一个对象
- 第二个参数即`accumulator`，类型为 `BiConsumer<R,? super T>` ，它的第一个参数是前面生成的 `ArrayList` 对象，第二个参数是stream中包含的元素，方法体就是把stream中的元素加入ArrayList
对象中。第二个方法被反复调用直到原stream的元素被消费完毕；
- 第三个参数即`combiner`，类型为 `BiConsumer<R, 
R>`，它的两个参数都是 `ArrayList` 类型的，方法体就是把第二个 `ArrayList` 全部加入到第一个中；

可见`reduce`和`collect`方法中后两个变量的区别：
- `accumulator`：收集器，对于规约来说，它需要对两个参数进行操作，如加和、比较等，并有一个返回结果；对于收集来说，它需要将一个参数加入到另外一个 `container` 里，没有返回值
- `combiner`：组合器，同上，对于规约来说，它需要对部分中间结果进行合并并返回最终的结果；对于收集来说，它需要对部分中间结果进行合并，没有返回值

> 也就是说 `collect` 需要自己提供返回结果的 `container` 对象的创建过程，而 `reduce` 
不需要，它是一步一步将中间结果计算后返回的，不计算到最后不知道这个结果是如何构造出来的。


该方法的参数较多，代码有点繁琐，其实 `collect` 还有另外一个重载方法:

#### 2. 一个参数版本

> <R, A> R collect(Collector<? super T, A, R> collector)

里面只有一个参数 `Collector` ，它是一个接口，看过它的定义就知道它暴露了几个方法，其中最重要的几个声明如下：

```java
    /**
     * A function that creates and returns a new mutable result container.
     *
     * @return a function which returns a new, mutable result container
     */
    Supplier<A> supplier();

    /**
     * A function that folds a value into a mutable result container.
     *
     * @return a function which folds a value into a mutable result container
     */
    BiConsumer<A, T> accumulator();

    /**
     * A function that accepts two partial results and merges them.  The
     * combiner function may fold state from one argument into the other and
     * return that, or may return a new result container.
     *
     * @return a function which combines two partial results into a combined
     * result
     */
    BinaryOperator<A> combiner();
    
    /**
     * Perform the final transformation from the intermediate accumulation type
     * {@code A} to the final result type {@code R}.
     *
     * <p>If the characteristic {@code IDENTITY_TRANSFORM} is
     * set, this function may be presumed to be an identity transform with an
     * unchecked cast from {@code A} to {@code R}.
     *
     * @return a function which transforms the intermediate result to the final
     * result
     */
    Function<A, R> finisher();
```

可以看到其实它的内部三个方法跟`collect`三个参数版本的中的三个参数： `supplier` 、 `accumulator` 、 `combiner` 
是一一对应的，只不过 `combiner` 是`BinaryOperator` 类型，而不是 `BiConsumer` 类型，这个参数类型和 `reduce` 
方法的三参数版本中的 `combiner` 是一样的。

那么上面复杂的三参数实现的收集集合信息的代码就可以变成下面这种简洁的写法：

```java
List<Integer> list = Stream.of(1, 2, 3, 4).filter(p -> p > 2).collect(toList());
```

 `toList`方法是 `Collectors` 类提供的一个工具方法，它默认返回一个 `Collector` 对象，让我们来看一下`toList` 方法是如何实现这个接口的：

```java
public static <T>
    Collector<T, ?, List<T>> toList() {
        return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                   (left, right) -> { left.addAll(right); return left; },
                                   CH_ID);
    }
```

- `supplier`：生成了一个可变集合 `ArrayList`
- `accumulator`：将每个元素加入到可变集合中 `List::add`
- `combiner`：将收集过程中产生的多个中间结果子集合进行合并，并返回结果，主要是并行流中会使用
### 并行流
对于串行流的方式，以下几种 `collect` 操作的结果是一样的：
```java
    // 正常写法
    Stream<Integer> stream = Stream.of(1, 2, 3);
    List<Integer> result = stream.collect(() -> new ArrayList<>(), (list, item) -> list.add(item), (one, two) -> one.addAll(two));
    System.out.println(result);
    
     // combiner方法随意写
    stream = Stream.of(1, 2, 3);
    result = stream.collect(() -> new ArrayList<>(), (list, item) -> list.add(item), (one, two) -> one.size());
    System.out.println(result);
    
    // supplier使用共享的变量
    List aa = new ArrayList<>(); 
    stream = Stream.of(1, 2, 3);
    result = stream.collect(() -> aa, (list, item) -> list.add(item), (one, two) -> one.addAll(two));
    System.out.println(result);
```

对于并行流方式，只有第一种方式可以成功：
```java
    // 正确写法
    Stream<Integer> stream = Stream.of(1, 2, 3);
    List<Integer> result = stream.parallel().collect(() -> new ArrayList<>(), (list, item) ->
            list.add(item), (one, two) -> one.addAll(two));
    System.out.println(result);
    
    // combiner会被调用，需要对子结果集进行聚合
    stream = Stream.of(1, 2, 3);
    result = stream.parallel().collect(() -> new ArrayList<>(), (list, item) -> list.add(item), (one, two) -> one.size());
    System.out.println(result); 
    // output: 1
    
    // 使用了共享变量，所以多线程情况下都是使用的一个集合，所以返回结果会非常多
    List aa = new ArrayList<>();
    stream = Stream.of(1, 2, 3);
    result = stream.parallel().collect(() -> aa, (list, item) -> list.add(item), (one, two) -> one.addAll(two));
    System.out.println(result);
    // output: 每次执行可能都不一样，还有可能报错IndexOutOfBoundException
```

对于最后一种方式，没有遵守`Collector`接口中对 `supplier` 方法的要求：`A function that creates and returns a new mutable result container.`

### 分组和分片

对具有相同特性的值进行分组是一个很常见的任务，`Collectors` 提供了一个 `groupingBy` 方法，方法签名为：

> \<T,K,A,D\> Collector<T,?,Map<K,D>> groupingBy(Function<? super T,? extends K> classifier, Collector<? super T,A,D> downstream)

- `classifier` ：分类器是一个`Function`，负责将数据按照某一规则进行分类并生成对应的 `Key`
- `downstream` ：下行流收集器是一个`Collector`，对分类后的数据集进行收集 `reduce` 操作

下面举几个例子：
1. 假如要根据年龄来分组并得到元素自身的类实例：
```java
Map<Integer, List<Person>> peopleByAge = people.stream().collect(groupingBy(p -> p.age, toList()));
```
因为是收集元素实例本身，所以直接使用 `toList` 即可。

2. 假如我想要根据年龄分组，年龄对应的键值，List存储 `Person` 的姓名:
```java
Map<Integer, List<String>> peopleByAge = people.stream().collect(groupingBy(p -> p.age, mapping((Person p) -> p.name, toList())));
```
由于并不是收集元素本身的类实例，需要做一层转换，这里用到了 `mapping` 方法。
`mapping` 方法即为对各 `组` 进行 `投影` 操作，和 `Stream` 的 `map` 方法基本一致。

注意到 `groupingBy` 方法的返回值其实也是一个 `Collector` ，而 `groupingBy` 的第二个参数也是 `Collector` 类型，那么说明我们可以使用多级的分组：

```java
Map<String, Map<String, List<Person>>> peopleByStateAndCity
              = personStream.collect(groupingBy(Person::getState, groupingBy(Person::getCity)));
            
```
在这里，第二个收集器我们称之为「下游收集器」，它是生成部分结果的配方，主收集器中会用到下游收集器。`groupingBy(classifier)` 内部使用了 `toList` 作为了 
`downstream` 的 `Collector` 。

再来看一个内置 `Collector` 的例子 —— `averagingInt` 方法，它内部直接实例化了 `Collector` 接口：
```java
public static <T> Collector<T, ?, Double>
    averagingInt(ToIntFunction<? super T> mapper) {
        return new CollectorImpl<>(
                () -> new long[2],
                (a, t) -> { a[0] += mapper.applyAsInt(t); a[1]++; },
                (a, b) -> { a[0] += b[0]; a[1] += b[1]; return a; },
                a -> (a[1] == 0) ? 0.0d : (double) a[0] / a[1], CH_NOID);
}
```
- `supplier`：初始值，这里是一个数组，第一个元素表示sum，第二个元素表示个数
- `accumulator`：一个 `BiConsumer` 函数，通过 `mapper` 对元素进行类型转化后与 `container` 计算，并累加个数
- `combiner`：一个 `BinaryOperator` 函数，这里对所有中间结果进行规约，加和，包括两个元素
- `finisher`：对最终结果进行计算平均值


#### reducing
3. 假如要根据姓名分组，并获取每个姓名下人的年龄总和：

注意这里并不是简单的返回某种数据收集后的结果，而是对这些数据进行某类计算操作后再返回，是不是想到了「规约」？
```java
Map<String, Integer> sumAgeByName = people.stream().collect(groupingBy(p -> p.name, reducing(0, (Person p) -> p.age, Integer::sum)));
/* 或者使用summingInt方法 */
sumAgeByName = people.stream().collect(groupingBy(p -> p.name, summingInt((Person p) -> p.age)));

```
这里使用的`reducing` 方法是`Collectors` 类提供的工具方法，它的签名和我们上面介绍的 `Stream` 的 `reduce` 方法在三个参数定义上有些不同：

> Collector<T, ?, U> reducing(U identity, Function<? super T, ? extends U> mapper, BinaryOperator<U> op) {

`identity`：循环计算的初始值，这个和 `reduce` 方法的第一个参数一样
`mapper`：类型转换器，将参数T转换为U类型，这个地方不太一样，主要是用于做类型转换，`reduce` 方法没这个概念，因为数据流在创建开始就是固定了类型的，无需转换
`op`：用于做 `reduce` 操作的 `BinaryOperator` 变量，这个 `op` 其实对应 `reduce` 方法中的二参数版本中的 `accumulator` 即对数据流中的数据进行聚合

来看一下它的实现，其实也是底层实例化了 `Collector` 接口，并将 `identity` 变成 `supplier` 每次返回的结果，并将 `mapper` 在 `accumulator` 
中对 `T` 类型元素 `t` 和 `U` 类型 `container` 进行运算：
```java
public static <T, U>
    Collector<T, ?, U> reducing(U identity,
                                Function<? super T, ? extends U> mapper,
                                BinaryOperator<U> op) {
        return new CollectorImpl<>(
                boxSupplier(identity),
                (a, t) -> { a[0] = op.apply(a[0], mapper.apply(t)); },
                (a, b) -> { a[0] = op.apply(a[0], b[0]); return a; },
                a -> a[0], CH_NOID);
    }
```

代码示例中给出的另外一个实现方法，即使用 `summingInt` 方法，它返回的是一个 `Collector` 实现，完成了规约的功能： 
```java
public static <T> Collector<T, ?, Integer>
    summingInt(ToIntFunction<? super T> mapper) {
        return new CollectorImpl<>(
                () -> new int[1],
                (a, t) -> { a[0] += mapper.applyAsInt(t); },
                (a, b) -> { a[0] += b[0]; return a; },
                a -> a[0], CH_NOID);
    }
```
它的参数是一个 `ToIntFunction` 实例，其实就是`reducing` 中的 `mapper` 参数，另外因为是通过 `collect` 方式实现的规约，所以需要提供：
- `supplier`，即`new int[1]`
- `accumulator`，即a[0] += mapper.applyAsInt(t)
- `combiner`，即a[0] += b[0]

再看一个例子如 `counting` 也是一个收集器：
```java
public static <T> Collector<T, ?, Long>
    counting() {
        return reducing(0L, e -> 1L, Long::sum);
    }
```
它通过 `reducing` 方法实现了收集，在这里它的三个参数为：
- `identity`：初始值，这里是0
- `mapper`：一个mapping函数，会对每一个元素执行转换，这里直接返回1
- `op`：一个 `BinaryOperator` 函数，这里对所有中间结果进行规约，加和



再来看一个例子，`averagingInt` 方法，它直接实现了 `Collector` 接口：
```java
public static <T> Collector<T, ?, Double>
    averagingInt(ToIntFunction<? super T> mapper) {
        return new CollectorImpl<>(
                () -> new long[2],
                (a, t) -> { a[0] += mapper.applyAsInt(t); a[1]++; },
                (a, b) -> { a[0] += b[0]; a[1] += b[1]; return a; },
                a -> (a[1] == 0) ? 0.0d : (double) a[0] / a[1], CH_NOID);
}
```
它的各个参数的分析：
- `supplier`：提供初始值，这里实例化一个int数组，长度为2，第一个元素表示sum，第二个元素表示个数
- `accumulator`：一个 `BiConsumer` 函数，通过 `mapper` 对元素进行类型转化后与 `container` 计算，并累加个数
- `combiner`：一个 `BinaryOperator` 函数，这里对所有中间结果进行规约，加和，包括两个元素
- `finisher`：对最终结果进行计算平均值

这个方法在「多维」的规约计算时很有用，比如`groupingBy` 或 `partitionBy`，在对一般的`Stream`进行规约时，直接用它自带的 `reduce` 方法即可，以下摘自 `JavaDoc` ：

> The reducing() collectors are most useful when used in a multi-level reduction, downstream of groupingBy or partitioningBy.
>
> To perform a simple reduction on a stream, use Stream.reduce(BinaryOperator) instead.
>
> For example, given a stream of Person, to calculate tallest person in each city:
>
> ```
>     Comparator<Person> byHeight = Comparator.comparing(Person::getHeight);
> 
>     Map<City, Person> tallestByCity
> 
>         = people.stream().collect(groupingBy(Person::getCity, reducing(BinaryOperator.maxBy(byHeight))));
> ```


`Collectors` 类提供很多类似的收集器：
> averagingDouble:求平均值，Stream的元素类型为double
> 
> averagingInt:求平均值，Stream的元素类型为int
> 
> averagingLong:求平均值，Stream的元素类型为long
> 
> counting:Stream的元素个数
> 
> maxBy:在指定条件下的，Stream的最大元素
> 
> minBy:在指定条件下的，Stream的最小元素
> 
> reducing: reduce操作
> 
> summarizingDouble:统计Stream的数据(double)状态，其中包括count，min，max，sum和平均。
> 
> summarizingInt:统计Stream的数据(int)状态，其中包括count，min，max，sum和平均。
> 
> summarizingLong:统计Stream的数据(long)状态，其中包括count，min，max，sum和平均。
> 
> summingDouble:求和，Stream的元素类型为double
> 
> summingInt:求和，Stream的元素类型为int
> 
> summingLong:求和，Stream的元素类型为long


### References

- https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#Reduction
- https://mohammadrasoolshaik.wordpress.com/2017/03/21/java-8-streams-collect-vs-reduce/

> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,
> 转载请保留原文链接.
