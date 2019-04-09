---
layout:     post
title:      "Optional Closure & @escaping"
subtitle:   "DailySwift"
date:       2019-04-09 21:52:00
author:     "Leppard"
header-img: "img/0409.jpg"
catalog: true
tags:
    - Swift
    - iOS
    - DailySwift
---


引子
在Swift的函数书写中，作为一等公民的闭包常被用作函数参数进行传递。而往往在参数闭包前能看到一个这样的修饰符：`@escaping`：

```
// Code1-1
// 来自官网的例子
var completionArr: [() -> Void] = []
func doSomething(completion: @escaping () -> Void) {
    completionArr.append(completion)
}
```

其实和`@escaping`配对的还有`@nonescaping`。下面简单介绍一下两者区别和使用场景，如果觉得介绍不够清晰可以查询官方文档或其他科普文；如果对这两个修饰符很熟悉的可以直接跳到正题部分。

### @escaping VS @nonescaping
简单来说两个修饰符描述了闭包的生命周期。

对于`@nonescaping`修饰的参数闭包，生命周期十分清晰：作为参数传入->函数执行该闭包->函数返回。由于闭包不会在函数返回之后执行，因此对于闭包中所捕获的所有变量，在函数作用域结束之前均能得到引用计数上的释放，内存管理十分方便和直观。

但是对于`@escaping`修饰的参数闭包情况会复杂一些。顾名思义，该参数闭包需要逃逸该函数，也就是生命周期需要超越过函数的作用域。就如Code1-1所示，在doSomething的函数作用域中，参数闭包completion被添加到了数组中，而在doSomething结束后，completion包括它所捕获的变量都依然需要存在，因为completion被completionArr持有了。在doSomething之外，我们依然需要执行comletion()，因此该completion的参数需要使用`@escaping`修饰，代表该闭包需要逃逸该函数作用域。由于该闭包生命周期变得复杂，故容易造成一些retain cycle的问题，需要显式在闭包中使用self，强制你仔细检查相互的持有关系。

## 正题
在Swift 3之后，默认参数闭包修饰符也从`@escaping`变为了`@nonescaping`，如果需要逃逸闭包需要自己手动声明为`@escaping`。关于`@escaping`和`@nonescaping`介绍到这里，来看看遇到的问题。
### 抛出问题
今天见到了这样一个报错：
```
// !!! Compile Error: @escaping attribute only applies to function types
public func download(url: String, completion: @escaping ((Bool)->Void)?) {
    // excute completion asynchronously when download.
}
```

场景如下：download函数接收一个闭包参数，这个闭包由于是在网络下载成功后异步执行。对于completion产生了如下两个要求：
1. 由于执行completion的时候download函数已经结束了。因此completion需要标记为`@escaping`以让其能在download函数生命周期之外能执行。
2. 业务方可能不需要在下载完成后执行回调，因此completion应该是一个Optional的值。

结合如上两点要求，我们想要声明一个escaping的optional闭包，但是书写出来之后发现编译器出现了报错：

> @escaping attribute only applies to function types

[Escaping Optional Closure](https://ws3.sinaimg.cn/large/006tNc79gy1g1wn8ywq9kj31ga08mglw.jpg)

### 问题解决
按照Xcode建议方式修改，去除了`@escaping`修饰符后，编译可正常通过。
因此在使用时，一个Optional的闭包不需要加`@escaping`或者`@nonescaping`的修饰符，直接声明即可。

### 原因分析
问题解决之外，我们难免会好奇，为什么对于Optional的闭包，默认修饰符是逃逸的？

我们还是从报错入手：
> @escaping attribute only applies to function types
编译器试图向你解释，@escaping只能修饰函数（闭包）类型，现在Optional的completion闭包已经不是该类型了。

恍然大悟，现在的completion参数是Optional类型，也就是`Enum`的枚举类型，是其中的`case .Some`里包含了一个闭包。也就是说，其实我们现在的参数类型严格来说是一个Enum，而不是闭包类型，因此无法用@escaping修饰。

然而，由于闭包被包裹在了Optional中，就带来了另一个疑问：在Optional中的闭包是@escaping的还是@nonescaping的？

为了解答这个问题，查阅官方文档后找到这样一句话：
> One way that a closure can escape is by being stored in a variable that is defined outside the function.

也就是说，如果闭包存储在了变量中，那么它默认就是可逃逸的。这其实很好理解，由于该闭包存储在了变量中，因此可能存在多次调用的可能性，切调用时机不可控，逃逸性只能是@escaping的。

而对于Optional的闭包来说也是一样，本质上是一个Enum存储了这个闭包，为了防止这个Optional稍后会再次使用到这个闭包，也将闭包修饰为了@escaping的了。

总结一下，闭包除了在函数参数位置上，其他所有的闭包默认都是逃逸的。闭包逃逸性默认情况如下：
* @nonescaping: 直接作为函数参数的闭包
* @escaping: 其他所有闭包，包括被赋值给变量，存在于Array、Optional中

## 尾声
由于开了博客一直不知道写些什么，因此把这次的小思考记录下来，作为”DailySwift”的开篇。从解决编译问题到尝试理解背后的设计原理，这其中的过程虽然有些费时，但却是一天最充满乐趣的时刻。


