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
3. `Executor`类
3. ...

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

我们既可以用`std`标准库的方式从`Promise`去创建一个`Future`，也可以通过`makeFuture`函数去创建`Future`。从这个创建出来的`Future`可以构建调用链，`thenValue`可以接受传统的可调用对象作为参数，而`then`则特殊一点，他的参数是`Try`，我们后面再谈，无论哪种方式他们的返回值都是一个新的`Future`。

最后获取`Future`里存储的数据的方式也特殊一点，和`std`略有不同，使用`value()`。`std`获取数据经常使用的`get`也可以继续使用，但只用于右值操作（我们的例子里可以改成`move(f2).get()`）。

细心的读者可以发现，我们在创建回调链的时候，使用了`move(f1)`, 直接使用`f1`可以吗？`folly`并不允许，整个chain都是基于右值的。


## Future
(TBD)
## Try & Executor
(TBD)

## Promise

## Continuation
(TBD)
# TBD





