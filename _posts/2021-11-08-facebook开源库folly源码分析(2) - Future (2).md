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

`Try`的作用有一点类似std标准库中的`_Result`(定义shard state里），实现了存储数据的功能，但显然清晰了很多。比起std使用static_cast强制转换成目标类型，Try直接就保存了类型信息。`_Result`广泛用于share state中，同样的，在folly中也有对应share state的设计，叫做`Core`对象，`Try`也广泛应用于`Core`中。关于`Core`我们后面会再来看。

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
  template <typename T>
  struct isTry : std::false_type {};

  template <typename T>
  struct isTry<Try<T>> : std::true_type {};

  ...

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

semifuture代表一个未来可访问的值(future value), 只有它有能力访问这个值, 很类似std标准库中的future。我们可以通过via操作来指定一个semifuture和对应的executor，去获得一个continable future。semifuture不会直接关联executor, 因此semifuture本身是无法支持链式调用的。

continuable future则是新推出的概念，顾名思义，他支持了continuation，因此可以链式调用。每个continuable future都绑定了一个executor, 我们可以利用then/collect/...去简洁地在不同的executor上去执行我们的并发逻辑。

executor的引入，让future可以不再局限于标准库的`std::thread`, 将两者合理的解耦了。

回到`folly`, `folly`定义了2种future: `SemiFuture`和`Future`。

comsumer侧(等同于`std`中的provider)通常生成的是`SemiFuture`, 而不是`Future`。我们需要使用`via()`函数去将`SemiFuture`转成`Future`, 来构建后续的回调链。`via()`函数会指定当前`Future`使用的executor, 实际上我们是通过executor去制定了未来future的执行行为。

举个例子，在我们最上面的例子里面，我们使用`getFuture()`去从`Promise`对象中生成了一个`Future`对象。看起来似乎我们并没有用到`SemiFuture`, 但实际上`getFuture()`函数内部首先调用`getSemiFuture()`函数去生成了一个`SemiFuture`对象，然后指定默认的`InlineExecutor`作为executor从`SemiFuture`对象生成`Future`对象。关于各种executor，我们后面再介绍。

在老的代码里，一般都用`makeFuture()`创建一个Future, 但这种写法在folly里已经被deprecated了，代替他的是`makeSemiFuture()`, folly推荐用semiFuture并配合via指定executor来使用future。不管哪一种方法，在底层都是new了一个`Core`对象，并用于创建对应的future。

`Future`和`SemiFuture`有着共同的基类`FutureBase`,其定义如下：
```
template <class T>
class FutureBase {
 protected:
  using Core = futures::detail::Core<T>;

 public:
  typedef T value_type;
  ...
  // shared core state object
  // usually you should use `getCore()` instead of directly accessing `core_`.
  Core* core_;
  ...
};
```
`FutureBase`主要的功能就是封装了`Core`对象，future和promise之间的数据共享都是依靠`Core`来工作的。这里存储的是core对象的指针，方便移动构造。

`FutureBase`作为基类，提供了很多通用的方法便于使用，其中其实大部分都基于Try。一些重要的方法如下:
|方法|返回值|说明|
|-|-|-|
|value|T&/T&&||
|result|Try<T>&/Try<T>&&||
|is_ready|bool||
|hasValue|bool||
|hasException|bool||
|getCore|Core&|获取core对象(如果不存在则抛异常)|
|getExecutor|Executor*|获取core对象的executor|

`SemiFuture`和`Future`都继承了`FutureBase`，从数据上来说，他们基本没有区别，他们的区别主要在于各自的行为上，作为子类，他们分别实现了不同的构造，赋值，移动和各自的特性。举个例子，只有`Future`支持Continuation。关于Continuation，后面我们专门来分析。
|功能|SemiFuture|Future|说明|
|-|-|-|-|
|construct from value|支持|支持||
|拷贝构造|不支持|不支持||
|移动构造|支持|支持||
|wait|支持|不支持|阻塞调用者线程直到Future对象ready|
|get/getTry|支持|支持|SemiFuture不支持getVia|
|via|支持，返回Future|支持|
|then/thenTry/thenError|不支持|支持|
|defer|支持|不支持||
|onError/onTimeout|不支持|支持||
|within|支持|支持||

`SemiFuture`和`Future`都可以通过makeFuture的方式来创建，他们也可以互相转换。 `SemiFuture`可以通过`toUnsafeFuture()`转成`Future`, `Future`当然也可以通过`semi()`转成`SemiFuture`。

在和`Promise`配合使用的时候，很多时候我们并不用特殊的区分这两者。我们既可以使用promise的`getSemiFuture()`方法去获得semiFuture，也可以通过`getFuture()`方法去直接获得future(默认使用了inline executor)。但这2个方法都被deprecated了，取代他们的是`folly::makePromiseContract()`。

当我们想获取future/semi future中的数据的时候，直接使用get就可以获得。实现的时候，则是通过获得底层的try，然后实现的。相比std而言，是简单很多了。举个例子。
```
template <class T>
T SemiFuture<T>::get() && {
  return std::move(*this).getTry().value();
}
```

总的来看，其实folly的future本身数据很简单，核心的设计主要在`Core`对象上。

### Core 
`Core`对象对应std中的shared state, 这是整个框架里最重要的类，他用来实现future和promise直接状态的共享和通知。
`Core`其实是一个子类，派生自代表数据的`ResultHolder<T>`（private继承）和代表共享状态的`CoreBase`(public继承)。

代表数据的`ResultHolder`非常简单，其实就是Try。
```
template <typename T>
class ResultHolder {
 protected:
  ResultHolder() {}
  ~ResultHolder() {}
  // Using a separate base class allows us to control the placement of result_,
  // making sure that it's in the same cache line as the vtable pointer and the
  // callback_ (assuming it's small enough).
  union {
    Try<T> result_;
  };
};
```

代表共享状态的`CoreBase`的实现就相对复杂。`CoreBase`类本身定义如下:
```
class CoreBase {
 protected:
  using Context = std::shared_ptr<RequestContext>;
  using Callback = folly::Function<void(
      CoreBase&, Executor::KeepAlive<>&&, exception_wrapper* ew)>;

  ...
 protected:
  CoreBase(State state, unsigned char attached);

  ...

  void detachOne() noexcept;

  void derefCallback() noexcept;

  union {
    Callback callback_;
  };
  std::atomic<State> state_;
  std::atomic<unsigned char> attached_;
  std::atomic<unsigned char> callbackReferences_{0};
  KeepAliveOrDeferred executor_;
  union {
    Context context_;
  };
  std::atomic<uintptr_t> interrupt_{}; // see InterruptMask, InterruptState
  CoreBase* proxy_;
};
```
`CoreBase`维护了3套数据：处理producer到consumer基于state的FSM(finite state machine, 有限状态机);从consumer到producer的interrupt(exception)异常流；和维护数据对象的同步和引用计数机制。

先来看state状态机，这也是最核心的部分, state对应的成员是`std::atomic<State> state_`。folly为`CoreBase`定义了一套状态码，具体如下，这些状态转移操作都是原子的。
```
enum class State : uint8_t {
  Start = 1 << 0,
  OnlyResult = 1 << 1,
  OnlyCallback = 1 << 2,
  OnlyCallbackAllowInline = 1 << 3,
  Proxy = 1 << 4,
  Done = 1 << 5,
  Empty = 1 << 6,
};
```
folly官方注释直接描述了状态机的转换图。
```
///   +----------------------------------------------------------------+
///   |                       ---> OnlyResult -----                    |
///   |                     /                       \                  |
///   |                  (setResult())             (setCallback())     |
///   |                   /                           \                |
///   |   Start --------->                              ------> Done   |
///   |     \             \                           /                |
///   |      \           (setCallback())           (setResult())       |
///   |       \             \                       /                  |
///   |        \              ---> OnlyCallback ---                    |
///   |         \           or OnlyCallbackAllowInline                 |
///   |          \                                  \                  |
///   |      (setProxy())                          (setProxy())        |
///   |            \                                  \                |
///   |             \                                   ------> Empty  |
///   |              \                                /                |
///   |               \                            (setCallback())     |
///   |                \                            /                  |
///   |                  --------> Proxy ----------                    |
///   +----------------------------------------------------------------+
```
1. Start: 初始状态。
2. OnlyResult: 设置了数据(result), 且从未被访问。
3. OnlyCallback: 类似OnlyResult。这里的callback其实指的是continuation中的func。
4. Done: 使用了链式调用后的状态。
5. Proxy: 这个状态比较特殊。我猜测是对应了callback里返回semiFuture的特殊场景的处理,最后把行为proxy到future，而不是semifuture上。（不是很确定，未来有时间的话会好好分析）
6. Empty: 已经proxy core之后的状态。
7. 其他的看图基本就明白了。

然后我们再来看看interupt, 他存储在成员`std::atomic<uintptr_t> interrupt_`里。interupt也定义了一套状态，如下:
```
  enum InterruptState : uintptr_t {
    InterruptInitial = 0x0u,
    InterruptHasHandler = 0x1u,
    InterruptHasObject = 0x2u,
    InterruptTerminal = 0x3u,
  };
```
虽然看起来状态有4个，但其实真正用的上的只有前2个。interupt主要用于consumer通过异常的方式通知到producer。异常机制的实现的代码比较直观。
```
void CoreBase::raise(exception_wrapper e) {
  if (hasResult()) {
    return;
  }
  auto interrupt = interrupt_.load(std::memory_order_acquire);
  switch (interrupt & InterruptMask) {
    case InterruptInitial: { // store the object
    ... 
    }
    case InterruptHasHandler: { // invoke the stored handler
      auto pointer = interrupt & ~InterruptMask;
      auto exchanged = interrupt_.compare_exchange_strong(
          interrupt, pointer | InterruptTerminal, std::memory_order_relaxed);
      if (!exchanged) { // ignore all calls after the first
        return;
      }
      auto handler = reinterpret_cast<InterruptHandler*>(pointer);
      handler->handle(e);
      return;
    }
    case InterruptHasObject: // ignore all calls after the first
      return;
    case InterruptTerminal: // ignore all calls after the first
      return;
  }
}
```

在使用的时候producer需要先注册handler，(比如promise的setInterruptHandler)。当consumer侧出现异常的时候, （可以通过future.raise()触发），会调用producer的handler处理。显然，对于hasObject和Terminal的状态，其实是没有什么意义的，因为这时候其实已经成功了，并不会有异常。
```
void Promise<T>::setInterruptHandler(F&& fn) {
  getCore().setInterruptHandler(static_cast<F&&>(fn));
}
```

最后是引用计数。`CoreBase`里维护了2套引用计数，`std::atomic<unsigned char> attached_`用来表示core的引用计数，`std::atomic<unsigned char> callbackReferences_`用来表示callback的引用计数。

在`Core`被构造的时候，(通过`Core`不允许直接创建对象，必须通过提供的make接口)，state和引用计数都会被初始化，根据情况的不同，引用的计数也不同。之前我们提到，Future对象里维护了一个Core指针，无论是semi future，还是future，在析构的时候，都会有相应的计数处理（逻辑细节不同）。

引用计数的增加，主要发生在`doCallback`方法，这个方法是很多Continuation操作的基础。而引用计数的减少，主要是通过一系列的tetach方法提供的。这部分比较复杂，涉及到很多edge case，笔者也没搞的很明白，等以后有时间了再分析。

除了这3套数据流外，`CoreBase`还维护了`KeepAliveOrDeferred`对象，这是`KeepAlive`或者`DeferredExecutor`的封装。等我们后面分析executor的时候再回来讲这个。`CoreBase`还有一个特殊的成员，`Context`，这是一个线程相关上下文的共享指针。这里就不细谈了。

说完了`CoreBase`，让我们回到本章的主角，`Core`。`Core`作为一个派生类，并没有自己特有的数据成员，但是提供了很多自己的方法，来满足框架的需求。比如我们之前提到的make,还有Continuation底层相关的`setCallback`和`setResult`。

关于`Core`的实现细节还有很多，留待以后吧。

## Promise
folly基于`folly::Try`实现了自己的promise。相对于std标准库，folly的实现非常清晰。
```
template <class T>
class Promise {
 public:
  ...
  /// Constructs a valid but unfulfilled promise.
  Promise();

 private:
  ...
  // Whether the Future has been retrieved (a one-time operation).
  bool retrieved_;
  ...
  // shared core state object
  // usually you should use `getCore()` instead of directly accessing `core_`.
  Core* core_;
  ...
};
```
实际上promise中只有2个成员，其中最主要的是shared state对象指针`core_`。基本上所有的promise的操作都是利用core完成的。
`Promise`实现了一些重要的方法，如下:
|方法|返回值|说明|
|-|-|-|
|getSemiFuture|SemiFuture<T>|deprecated. 推荐使用makePromiseContract.|
|getFuture|Future<T>|同上|
|setException|void|填充异常到promise里|
|setInterruptHandler|void|设置异常处理handler, 如果future抛异常，producer可以通过这个处理|
|setValue|void|填充数据到promise里|
|setTry|void|填充try到promise里，比较通用的方法|
|isFulfilled|bool|是否promise已经被填充了|
之前我们提到了，`getSemiFuture`和`getFuture`已经被deprecated了，folly推荐使用`makePromiseContract`。其实它只是一个简单的封装，用来规范我们对promise/future的使用。
```
template <class T>
std::pair<Promise<T>, SemiFuture<T>> makePromiseContract() {
  auto p = Promise<T>();
  auto f = p.getSemiFuture();
  return std::make_pair(std::move(p), std::move(f));
}
```
promise的核心操作是set系列函数，都是通过core指针的`setResult`实现的。这里就不赘述了。

### `SharedPromise`

### promise in fibers
fibers定义了自己的promise。
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
可以看到promise的成员只有2个，一个保存数据的`Try`对象`value`, 一个是Baton指针。

(TBD)
#### `Fiber`
fiber是一个比较新的名词，中文称作纤程，他和协程`coroutine`其实是相同的概念，都代表用户态的轻量级异步任务。多个fiber可以运行在一个系统线程上，由fiber manager去进行调度，切换合适的fiber上下文。这种调度是非常轻量级的，按照facebook的说法，1s可以切换2亿次的样子。

facebook在`folly`库里实现了自己的`Fiber`, 在`folly/fibers`目录下。主要包括：
1. `Fiber` : 纤程对象，每个`Fiber`对象都关联着一个`FiberManager`, 且只能被一个task执行一次。
2. `FiberManager` : Fiber调度器。
3. `Baton` : low level的异步信号量，`Baton`被用于fiber task之间的互相通知和等待。`Baton`两个最基本的操作是`wait()`和`post()`。

关于`Fiber`我们后面单独再分析源码实现。在这里提到，主要是因为fibers的`Future`的底层实现用到了baton。当我们泛化`Promise`的时候，第二个模板参数就是baton。

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


## Continuation
前面已经提到，folly和std最大的区别就是支持了continuation, 通俗地说，就是链式调用。

让我们来看一个简单的例子。
```
    auto f1 = makeFuture(1);
    auto f2 = move(f1).thenValue(foo1).then(foo2).thenValue(foo3);
```
我们创建了一个semi future f1,

## Executor
(TBD)



## ExceptionWrapper
(TBD)

# TBD





