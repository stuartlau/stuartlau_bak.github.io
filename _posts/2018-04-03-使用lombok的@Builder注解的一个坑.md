---
layout:     post
title:      使用lombok的@Builder的注解的一个坑
subtitle:   总有一个坑你要跳进去
date:       2018-04-03
author:     L.S
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - TroubleShooting
---    
# Background
在增加某feature的时候需要给某类添加一个小方法，该方法使用了内部成员变量，并且该成员变量在定义的时候就已经实例化：<br>
    
    Map<String, Object> map = Maps.newHashMap()


结果在调用map.contains()方法的时候报**NullPointerException**



# Analysis
这个类本身是使用lombok来实现的，并且使用了`@Builder`注解。

看一下它的`Java Doc`：
```
@Target(value={TYPE,METHOD,CONSTRUCTOR})
@Retention(value=SOURCE)
public @interface Builder
The builder annotation creates a so-called 'builder' aspect to the class that is annotated or the class that contains a member which is annotated with @Builder.
If a member is annotated, it must be either a constructor or a method. 
If a class is annotated, then a private constructor is generated with all fields as arguments (as if @AllArgsConstructor(AccessLevel.PRIVATE) is present on the class), 
and it is as if this constructor has been annotated with @Builder instead.
The effect of @Builder is that an inner class is generated named TBuilder, with a private constructor. 
Instances of TBuilder are made with the method named builder() which is also generated for you in the class itself (not in the builder class).
The TBuilder class contains 1 method for each parameter of the annotated constructor / method (each field, when annotating a class), which returns the builder itself. 
The builder also has a build() method which returns a completed instance of the original type, 
created by passing all parameters as set via the various other methods in the builder to the constructor or method that was annotated with @Builder. 
The return type of this method will be the same as the relevant class, unless a method has been annotated, in which case it'll be equal to the return type of that method.
```

发现它的实现方式是会对标注这个注解的类的所有成员变量，所以在使用`@Builder`构建的时候如果不显式的对某变量赋值的话默认就是null，因为这个变量此时是在Builder
类里的，通过调用build()方法生成具体T类则是通过私有构造函数来实例化，默认是全参数的构造函数，所以上面的map是作为其中的一个参数的，最终它就没有被赋值。



`@Builder`默认的实现方式是在类上添加@AllArgsConstructor(access = AccessLevel.PACKAGE)

# How-to Use Default Value
这里分享两种方法可以用自己定义的默认值而不被@Builder复写

## 1.自己实现minimal的Builder类

这里我们的类是GrpcHandlerContext，`@Builder`标注在该类上，默认一个private的属性**map**

    private Map<String, Object> map;
    public static class GrpcHandlerContextBuilder{
        private Map<String, Object> map = Maps.newHashMap();
    }

这样lombok生成的Builder类跟我们定义的类同名，就不会覆盖里面已经实例化的属性了。

## 2.用@Builder.Default来标识Field

    @Builder.Default
    private Map<String, Object> map = Maps.newHashMap();

这个注解是在1.16.16之后才有的

# Conclusion
如果使用`@Builder`的话切记所有私有全局变量都是需要显式赋值的，否则就是Null，不管你在原生T类中是否实例化，最终都是要被Builder的build()方法来重新实例化的

如果使用了`@Builder`注解就不要使用其他@ConstructorArgs相关的注解，这从设计模式上是冲突的



# Reference
- https://projectlombok.org/features/Builder

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
