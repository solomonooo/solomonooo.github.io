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

我们既可以用`std`标准库的方式从`Promise`去创建一个`Future`，也可以通过`makeFuture`函数去创建`Future`。从这个创建出来的`Future`可以构建调用链，`thenValue`可以接受传统的可调用对象作为参数，而`then`则特殊一点，他的参数是`Try`，（我们这里也可以使用`thenTry`），我们后面再谈，无论哪种方式他们的返回值都是一个新的`Future`。

最后获取`Future`里存储的数据的方式也特殊一点，和`std`略有不同，使用`value()`。`std`获取数据经常使用的`get`也可以继续使用，但只用于右值操作（我们的例子里可以改成`move(f2).get()`）。

细心的读者可以发现，我们在创建回调链的时候，使用了`move(f1)`, 直接使用`f1`可以吗？`folly`并不允许，整个chain都是基于右值的。

## Fiber, Baton & other basic object



## Try



## Future
folly定义了2种future类: `Future`和`SemiFuture`。

comsumer侧(等同于`std`中的provider)通常生成的是`SemiFuture`, 而不是`Future`。我们需要使用`via()`函数去将`SemiFuture`转成`Future`, 来构建后续的回调链。`via`函数会指定当前`Future`使用的executor, 实际上我们是通过executor去制定了未来future的执行行为。

举个例子，在我们上面的例子里面，我们使用`getFuture()`去从`Promise`对象中生成了一个`Future`对象。看起来似乎我们并没有用到`SemiFuture`, 但实际上`getFuture()`函数内部首先调用`getSemiFuture()`函数去生成了一个`SemiFuture`对象，然后指定默认的`InlineExecutor`作为executor从`SemiFuture`对象生成`Future`对象。关于各种executor，我们后面再介绍。


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

  ~Promise();

  // not copyable
  Promise(const Promise&) = delete;
  Promise& operator=(const Promise&) = delete;

  // movable
  Promise(Promise&&) noexcept;
  Promise& operator=(Promise&&);

  /** Fulfill this promise (only for Promise<void>) */
  void setValue();

  /** Set the value (use perfect forwarding for both move and copy) */
  template <class M>
  void setValue(M&& value);

  /**
   * Fulfill the promise with a given try
   *
   * @param t A Try with either a value or an error.
   */
  void setTry(folly::Try<T>&& t);

  /** Fulfill this promise with the result of a function that takes no
    arguments and returns something implicitly convertible to T.
    Captures exceptions. e.g.

    p.setWith([] { do something that may throw; return a T; });
  */
  template <class F>
  void setWith(F&& func);

  /** Fulfill the Promise with an exception_wrapper, e.g.
    auto ew = folly::try_and_catch([]{ ... });
    if (ew) {
      p.setException(std::move(ew));
    }
    */
  void setException(folly::exception_wrapper);

  /**
   * Blocks task execution until given promise is fulfilled.
   *
   * Calls function passing in a Promise<T>, which has to be fulfilled.
   *
   * @return data which was used to fulfill the promise.
   */
  template <class F>
  static value_type await_async(F&& func);

#if !defined(_MSC_VER)
  template <class F>
  FOLLY_ERASE static value_type await(F&& func) {
    return await_sync(static_cast<F&&>(func));
  }
#endif

 private:
  Promise(folly::Try<T>& value, BatonT& baton);
  folly::Try<T>* value_;
  BatonT* baton_;

  void throwIfFulfilled() const;

  template <class F>
  typename std::enable_if<
      std::is_convertible<invoke_result_t<F>, T>::value &&
      !std::is_same<T, void>::value>::type
  fulfilHelper(F&& func);

  template <class F>
  typename std::enable_if<
      std::is_same<invoke_result_t<F>, void>::value &&
      std::is_same<T, void>::value>::type
  fulfilHelper(F&& func);
};
```

### `SharedPromise`

## Continuation
(TBD)
# TBD





