---
layout: mypost
title: Facebook开源库folly源码分析(2) - Future
categories: [folly]
---
# folly
facebook的folly提供了自己的future/promise实现，在std标准库的基础上更进了一步。如同facebook自己在官方md上说的，folly不是对std的替代方案，而是在特定场景下的高性能优化。

folly的future一个非常重要的改进是通过`thenValue`和`thenTry`（以及其他函数）支持了future的链式调用，极大提升了异步编程的可读性和可维护性。我们后面会重点看下这部分。

folly的future/promise模型在`folly`/`futures`目录下。主要的类包括：
1. `Future`
2. `Promise`
3. `Executor`
4. ...

因为folly本身也在不断发展，本文以`v2021.10.04.00`版本为准，可能在某些类和函数的实现上和其他文章略有不同。

## folly/Future的简单示例
```
Future<int> foo1(int x) {
    std::cout << "foo1: try to add 1 for " << x << endl;
    return makeFuture<int>(x+1);
}

Future<int> foo2(const Try<int>& t) {
    std::cout << "foo2: try to add 1 for " << t.value() << endl;
    return makeFuture<int>(t.value()+1);
}

    ...

    cout << "make promise" <<endl;
    Promise<int> p;
    auto f1 = p.getFuture();
    //auto f1 = makeFuture<int>(1);
    auto f2 = move(f1).thenValue(foo1).then(foo2);
    cout << "future chain made" <<endl;

    cout << "fulfilling promise" <<endl;
    p.setValue(1);
    cout << "promise/future fulfilled, res = " <<f2.value()<<endl;
	...

}
```
在这个简单的例子里，我们创建了一个future的调用链，包括`f1`和`f2`，分别执行加一的操作。

我们既可以用`std`标准库的方式从`Promise`去创建一个`Future`，也可以通过`makeFuture`函数去创建`Future`。从这个创建出来的`Future`可以构建调用链，`thenValue`可以接受传统的可调用对象作为参数，而`then`则特殊一点，他的参数是`Try`，（我们这里也可以使用`thenTry`），我们后面再谈，无论哪种方式他们的返回值都是一个新的`Future`。

最后获取`Future`里存储的数据的方式也特殊一点，和`std`略有不同，使用`value()`。`std`获取数据经常使用的`get`也可以继续使用，但只用于右值操作（我们的例子里可以改成`move(f2).get()`）。

细心的读者可以发现，我们在创建回调链的时候，使用了`move(f1)`, 直接使用`f1`可以吗？`folly`并不允许，整个chain都是基于右值的。

## `Fiber`, `Baton` & other basic object
### `Fiber`
fiber是一个比较新的名词，中文称作纤程，他和协程`coroutine`其实是相同的概念，都代表用户态的轻量级异步任务。多个fiber可以运行在一个系统线程上，由fiber manager去进行调度，切换合适的fiber上下文。这种调度是非常轻量级的，按照facebook的说法，1s可以切换2亿次的样子。

facebook在`folly`库里实现了自己的`Fiber`, 在`folly/fibers`目录下。主要包括：
1. `Fiber` : 纤程对象，每个`Fiber`对象都关联着一个`FiberManager`, 且只能被一个task执行一次。
2. `FiberManager` : Fiber调度器。
3. `Baton` : low level的异步信号量，`Baton`被用于fiber task之间的互相通知和等待。`Baton`两个最基本的操作是`wait()`和`post()`。

关于`Fiber`我们后面单独再分析源码实现。在这里提到，主要是因为`Future`的底层实现用到了baton。当我们泛化`Promise`的时候，第二个模板参数就是baton。

一个简单的`Fiber`使用示例。
```
    folly::EventBase evb;
    folly::fibers::Baton baton;
    auto& fibermgr = folly::fibers::getFiberManager(evb);
    fibermgr.addTask([&](){
        std::cout << "Task 1: start" << std::endl;
        baton.wait();
        std::cout << "Task 1: after baton.wait()" << std::endl;
    });
    fibermgr.addTask([&](){
        std::cout << "Task 2: start" << std::endl;
        baton.post();
        std::cout << "Task 2: after baton.post()" << std::endl;
    });

    evb.loop();
```

## Try
`Try`是对`Future`中存储的数据、异常或nothing的封装, 被大量运用于`Future`的底层和接口，虽然大多数情况下我们并不需要自己去创建`Try`对象，但熟悉Try是熟练掌握`Future`的必要前提。

我们先来看`Try`的实现。
```
template <class T>
class Try {
  static_assert(
      !std::is_reference<T>::value, "Try may not be used with reference types");

  enum class Contains {
    VALUE,
    EXCEPTION,
    NOTHING,
  };
  ...

  Contains contains_;
  union {
    T value_;
    exception_wrapper e_;
  };
};
```
可以显然看到，`Try`是通过来enum `Contains`来区分存储数据的类型的，具体的数据则存储在union中，共享同一份空间。

`Try`的作用有一点类似std标准库中的shard state，实现了shared state存储数据的功能，但显然简洁干净了很多。

`Try`的关键函数有(对于相似的函数族，只列出最常用的一个)：
|方法|返回值|说明|
|-|-|-|
|`value()`|`T&`/`T&&`|获取`Try`存储的值(的引用)。如果存储的并不是值类型，则抛出异常。|
|`operator*`/`operator->`|和`value()`用处一样，都可以用于获取存储值的引用/指针|
|`has_value()`|`bool`|判断是否存储的是值|
|`hasException()`|`bool`|判断是否存储的是异常|
|`exception()`|`exception_wrapper`|获取存储异常的wrapper封装。关于`exception_wrapper`我们后面再说。|
|`withException(F func)`|`bool`|如果存储的是异常，则执行func F|
|`get()`|`T`/`Try<T>`|获取`Try`存储的值，或者`Try`本身，这里利用了`enable_if`来实现模板的多态。|

`get()`函数的实现值得我们学习一下。`enable_if`骚操作。
```
  template <bool isTry, typename R>
  typename std::enable_if<isTry, R>::type get() {
    return std::forward<R>(*this);
  }

  template <bool isTry, typename R>
  typename std::enable_if<!isTry, R>::type get() {
    return std::forward<R>(value());
  }
```
关于`enable_if`，有一篇文章可以学习一下。https://blog.csdn.net/jeffasd/article/details/84667090

## Future
按照facebook官方paper对于future的定义，future被分为了2种，semifuture和continuable future。( http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0904r0.pdf)

semiuture代表一个未来可访问的值(future value), 只有它有能力访问这个值, 很类似std标准库中的future。我们可以通过via操作来指定一个semifuture和对应的executor，去获得一个continable future。semifuture不会直接关联executor, 因此semifuture本身是无法支持链式调用的。

continuable future则是新推出的概念，顾名思义，他支持了continuation，因此可以链式调用。每个continuable future都绑定了一个executor, 我们可以利用then/collect/...去简洁地在不同的executor上去执行我们的并发逻辑。

executor的引入，让future可以不再局限于标准库的`std::thread`, 将两者合理的解耦了。

回到`folly`, `folly`定义了2种future: `SemiFuture`和`Future`。

comsumer侧(等同于`std`中的provider)通常生成的是`SemiFuture`, 而不是`Future`。我们需要使用`via()`函数去将`SemiFuture`转成`Future`, 来构建后续的回调链。`via()`函数会指定当前`Future`使用的executor, 实际上我们是通过executor去制定了未来future的执行行为。

举个例子，在我们最上面的例子里面，我们使用`getFuture()`去从`Promise`对象中生成了一个`Future`对象。看起来似乎我们并没有用到`SemiFuture`, 但实际上`getFuture()`函数内部首先调用`getSemiFuture()`函数去生成了一个`SemiFuture`对象，然后指定默认的`InlineExecutor`作为executor从`SemiFuture`对象生成`Future`对象。关于各种executor，我们后面再介绍。


(TBD)
## Try & Executor
(TBD)

## Promise
folly基于`folly::Try`实现了自己的promise。相对于std标准库，folly的实现非常清晰。
```
template <typename T, typename BatonT = Baton>
class Promise {
 public:
  typedef T value_type;
  typedef BatonT baton_type;
  ...

 private:
  folly::Try<T>* value_;
  BatonT* baton_;
  ...

};
```
(TBD)

### `SharedPromise`

## Continuation
(TBD)

## ExceptionWrapper
(TBD)

# TBD





