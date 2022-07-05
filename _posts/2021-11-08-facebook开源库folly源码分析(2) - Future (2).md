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

### Try的实现原理
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

### future的使用示例
future提供了`result()`和`value()`方便获取其中的数据或者异常，本文开头的简单示例中就展示了如何利用`value()`获取数据。更多的高级特性主要还是实现了continuation。关于这部分的使用，可以直接参考Continuation这一章。

### future的实现原理
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
|via|支持，返回Future|支持|可以指定executor|
|then/thenTry/thenError|不支持|支持||
|defer|支持|不支持||
|onError/onTimeout|不支持|支持||
|within|支持|支持||

`SemiFuture`和`Future`都可以通过makeFuture的方式来创建，他们也可以互相转换。 `SemiFuture`可以通过`toUnsafeFuture()`转成`Future`, `Future`当然也可以通过`semi()`转成`SemiFuture`。

在和`Promise`配合使用的时候，很多时候我们并不用特殊的区分这两者。我们既可以使用promise的`getSemiFuture()`方法去获得semiFuture，也可以通过`getFuture()`方法去直接获得future(默认使用了inline executor)。但这2个方法都被deprecated了，取代他们的是`folly::makePromiseContract()`。举个例子:
```
    auto [p, f] = folly::makePromiseContract<int>(&InlineExecutor::instance());
    cout<<"is future ready ? "<<f.isReady()<<endl;
    p.setValue(3);
    cout<<"is future ready ? "<<f.isReady()<<endl;
    cout<<"final f = "<<f.value()<<endl;
```

当我们想获取future/semi future中的数据的时候，直接使用get就可以获得。实现的时候，则是通过获得底层的try，然后实现的。相比std而言，是简单很多了。举个例子。
```
template <class T>
T SemiFuture<T>::get() && {
  return std::move(*this).getTry().value();
}

template <class T>
Try<T> SemiFuture<T>::getTry() && {
  wait();
  auto future = folly::Future<T>(this->core_);
  this->core_ = nullptr;
  return std::move(std::move(future).result());
}

template <class T>
Try<T>& FutureBase<T>::result() & {
  return getCoreTryChecked();
}

template <typename Self>
static decltype(auto) getCoreTryChecked(Self& self) {
  auto& core = self.getCore();
  if (!core.hasResult()) {
    throw_exception<FutureNotReady>();
  }
  return core.getTry();
}
```
通过一系列的调用，最后调用了core的getTry去获取数据。总的来看，其实folly的future本身数据很简单，核心的设计主要在`Core`对象上。

这里需要注意的是，在`getTry`的时候，首先调用了`wait()`来等待数据ready，这里其实是一个非常重要的设计，可以说，基本所有future的异步操作，最后都依赖于获取数据时的wait，cosumer会在这里等待数据的处理完成。我们后面再来说wait。

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

`Core`提供了一些重要的操作数据的方法，被广泛用于promise/future中，基本如下：

|方法|返回值|说明|
|-|-|-|
|make|Core*|创建一个core对象，主要是内部使用.|
|getTry|Try<T>&|comsumer使用，获取数据（或者异常）|
|setResult|Try<T>&|producer使用，设置数据（或者异常）|
|setCallback|void|顾名思义|

先来看`setResult`，这是填充数据或异常的底层操作。例如在promise调用`setValue`填充数据时，实际上底层调用的就是core的`setResult`。`setResult`先设置数据，然后内部调用了基类的`setResult_()`去同步状态和处理callback，这里主要是使用了上面提到的atomic state状态机。
```
  void setResult(Try<T>&& t) {
    setResult(Executor::KeepAlive<>{}, std::move(t));
  }

  void setResult(Executor::KeepAlive<>&& completingKA, Try<T>&& t) {
    ::new (&this->result_) Result(std::move(t));
    setResult_(std::move(completingKA));
  }
```
然后来看`getTry`，这是获取数据或异常的底层操作。例如在future调用`get()`的时，实际上底层调用的就是core的`getTry`。它的实现并不复杂，从core->result_中获取数据。
```
  Try<T>& getTry() {
    DCHECK(hasResult());
    auto core = this;
    while (core->state_.load(std::memory_order_relaxed) == State::Proxy) {
      core = static_cast<Core*>(core->proxy_);
    }
    return core->result_;
  }
```

关于`Core`的实现细节还有很多，留待以后吧。(TBD)

### wait
future的很多操作，都需要我们等待异步操作完成，数据被填充后继续执行，这个机制是在`get()`的时候调用`wait()`实现的，其底层使用了`Baton`。
```
template <class T>
Future<T>&& Future<T>::wait() && {
  futures::detail::waitImpl(*this);
  return std::move(*this);
}
```
可以看到`wait`直接调用了`waitImpl`,我们再来看`waitImpl`的实现。
```
template <class FutureType, typename T = typename FutureType::value_type>
void waitImpl(FutureType& f) {
  //一些优化操作
  ...
  Promise<T> promise;
  auto ret = convertFuture(promise.getSemiFuture(), f);
  FutureBatonType baton;
  f.setCallback_([&baton, promise = std::move(promise)](
                     Executor::KeepAlive<>&&, Try<T>&& t) mutable {
    promise.setTry(std::move(t));
    baton.post();
  });
  f = std::move(ret);
  baton.wait();
  assert(f.isReady());
}
```
可以看到这里使用baton来完成等待的操作, baton是folly提供的通知等待机制，这里就不详细解释了。简单来说，baton的`wait()`会等待通知，直到`post()`被调用为止。可以看到这里注册了一个callback，在数据或异常被填充后调用，最终通知回waitImpl里。

## Promise
在定义上，folly的promise和std没有本质区别，其代表producer端数据的opreation接口。
### promise的使用示例
对于promise而言，我们既可以填充数据，也可以填充异常。promise重载了很多版本的set函数方便来使用，我们只挑选经典的几个看下。

填充数据的例子。
```
    Promise<int> p;
    cout<<"is promise fullfilled ? "<<p.isFulfilled()<<endl;
    auto f = p.getFuture();
    cout<<"is future ready ? "<<f.isReady()<<endl;
    p.setValue(123);
    cout<<"is future ready ? "<<f.isReady()<<endl;
    cout<<"f = "<<f.value()<<endl;
```
填充异常的例子：
```
    Promise<int> p;
    cout<<"is promise fullfilled ? "<<p.isFulfilled()<<endl;
    auto f = p.getFuture();
    cout<<"is future ready ? "<<f.isReady()<<endl;
    p.setException(std::runtime_error("my exception"));
    cout<<"is future ready ? "<<f.isReady()<<endl;
    cout<<"has exception ?  "<<f.hasException()<<endl;
    //when call f.value(), throw exception.
```
promise其实还可以直接传入无参函数来填充数据。
```
    Promise<int> p;
    auto f = p.getFuture();
    p.setWith([]{
        return 123;
    });
```
### promise的实现原理
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
Promise的实现保证了`getSemiFuture`和`GetFuture`都只可以被调用一次, 如果非法调用，则会抛出异常`folly::FutureAlreadyRetrieved`。另外，`getSemiFuture`和`getFuture`已经被deprecated了，folly推荐使用`makePromiseContract`。其实它只是一个简单的封装，用来规范我们对promise/future的使用。
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
folly也提供了shared promise，和std的实现类似，`SharedPromise`可以通过`getFuture()`获取多次future。

实际上`SharedPromise`是通过封装`Promise`来实现的，其内部维护了一个promise的vector, 通过mutex来解决并发问题。

这里我们就不详细分析了。

### promise in fibers 
folly在fibers中，也定义了一个`Promise`, 服务于fibers。
`fibers::Promise`和原版的`Promise`类似，也提供了`setValue`/`setTry`这种填充数据的方法，但并不能extract出future对象来。他主要配合`await_async`来实现异步task。
`fibers::Promise`的实现并不复杂，但他的泛化需要2个参数，数据类型T和Baton。鉴于和fibers还有baton的关系比较紧密，等以后我们分析fibers的时候再来看看。

#### `Fiber`
fiber是一个比较新的名词，中文称作纤程，他和协程`coroutine`其实是相同的概念，都代表用户态的轻量级异步任务。多个fiber可以运行在一个系统线程上，由fiber manager去进行调度，切换合适的fiber上下文。这种调度是非常轻量级的，按照facebook的说法，1s可以切换2亿次的样子。

facebook在`folly`库里实现了自己的`Fiber`, 在`folly/fibers`目录下。主要包括：
1. `Fiber` : 纤程对象，每个`Fiber`对象都关联着一个`FiberManager`, 且只能被一个task执行一次。
2. `FiberManager` : Fiber调度器。
3. `Baton` : low level的异步信号量，`Baton`被用于fiber task之间的互相通知和等待。`Baton`两个最基本的操作是`wait()`和`post()`。

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
前面已经提到，folly和std最大的区别就是支持了continuation, 通俗地说，就是链式调用。folly提供了很多方法来实现continuation，一部分是future提供的方法，一部分是全局方法。

重要的方法如下：

|方法|说明|
|-|-|
|via|指定executor执行|
|then/thenTry/thenValue|future complete时执行|
|thenInline/thenTryInline/thenValueInline|同上,只是使用同一executor|
|thenError|当future有exception时执行|
|onError/onTimeout/ensure|很直观的含义，onError已经被thenError代替|
|within/delayed|加上时间控制的continuation|
|get/wait|其实不属于continuation，主要用于等待future被fullfilled|
|filter|用于filter，不满足条件的值会抛异常|
|mapValue/mapTry/reduce|没什么好说的,实现了map-reduce|
|collectAll/collectAllUnsafe|所有的输入future complete才继续执行,需要特殊注意的是返回值是semi future, unsafe版本才是future|
|collectAny|任意一个future complete就继续执行|
|collectN|没什么好说的|
|collect|等待输入future complete直到异常发生|
|window||
|whileDo||

上面列的每个方法，都有很多重载的版本来方便使用，其中then系列和collect系列是使用最广泛的方法。我们可以通过一系列例子来看如何使用。

这是一个比较经典的级联场景。
```
    cout << "make future chain" <<endl;
    auto f1 = makeFuture(1);
    auto f2 = move(f1).thenValue(foo1).then(foo2).thenValue(foo3);
    cout << "future chain made" <<endl;
```
当我们需要处理一些异常的时候，可以使用thenError去处理异常。当输入的future被正常填充数据时，thenError不会被触发，但一旦异常发生，就会进入thenError的处理逻辑。
```
    auto f1 = makeFuture<int>(1);
    auto f2 = move(f1).thenValue([](int x){
        //some unexpected happened.
        throw std::runtime_error("oh no!");
        return makeFuture<int>(x+1);
    })
    .thenError([](const auto& e){
        cout<<"exception : "<<e.what()<<endl;
        return -1;
    });
    cout << "future fulfilled, res = " <<f2.value()<<endl;

    exception : std::runtime_error: oh no!
    future fulfilled, res = -1
```
其实thenError非常类似传统的异常处理机制，thenError相当于catch，可以用来捕获异常（当然也可以继续抛出异常），folly也同时提供了类似finally的ensure函数。我们来看一个复杂的例子。
```
    auto f1 = makeFuture<int>(1);
    auto f2 = move(f1).thenValue([](int x){
        throw std::runtime_error("oh no!");
        return makeFuture<int>(x+1);
    })
    .thenError([](const auto& e){
        cout<<"exception : "<<e.what()<<endl;
        throw std::invalid_argument("oh no again!");
        return -1;
    })
    .thenError([](const auto& e){
        cout<<"exception : "<<e.what()<<endl;
        return -2;
    })
    .ensure([](){
        cout<<"my final work!"<<endl;
    });

    cout << "future fulfilled, res = " <<f2.value()<<endl;
```

仅仅是支持级联是无法满足复杂的场景的，folly还提供了一些高级方法去处理并行task。例如，collect系列函数，其中使用最多的是`collectAll`。`collectAll`会等待直到所有的输入future都complete。需要注意的是，`collectAll`的输出是semi future，我们可以通过toUnsafeFuture()来转换成future，或者直接使用`collectAllUnsafe`。
```
    std::vector<Future<int>> fv;
    for (int i = 0; i < 10; i++){
        fv.emplace_back(makeFuture(i));
    }

    auto f = folly::collectAll(fv)
        .toUnsafeFuture()
        .thenValue([](const std::vector<Try<int>>& trys){
            int res = 0;
            for (size_t i = 0; i < trys.size(); i++){
                int cur = trys[i].value();
                cout<<"f"<<i<<" = "<<cur<<endl;
                res += cur;
            }
            return makeFuture(res);
        });

    cout<<"is future ready ? "<<f.isReady()<<endl;
    cout<<"final f = "<<f.value()<<endl;
```
`collectAny`和`collectAll`功能类似,但他只会等待到input future中有一个complete就可以。需要注意的是他们的输出其实略有区别。`collectAll`的输出是一个Try的vector的集合，但`collectAny`的输出却是一个pair。很合理。
```
    std::vector<Future<int>> fv;
    for (int i = 0; i < 10; i++){
        fv.emplace_back(makeFuture(i));
    }

    auto f = folly::collectAny(fv)
        .toUnsafeFuture()
        .thenValue([](const std::pair<size_t, Try<int>>& trys){
            int res = trys.second.value();
            cout<<"f"<<trys.first<<" = "<<res<<endl;
            return makeFuture(res);
        });

    cout<<"is future ready ? "<<f.isReady()<<endl;
    cout<<"final f = "<<f.value()<<endl;
```
出人意料的，folly其实还提供了map/reduce方法。

`mapValue`的输入future的vector，其中每个元素都会被被传入的func参数处理成新的future。而`reduce`的输入同样是future的vector，但输出则是一个单独的future。举个例子：
```
    std::vector<Future<int>> fv;
    for (int i = 0; i < 10; i++){
        fv.emplace_back(makeFuture(i));
    }

    std::vector<Future<int>> fv2 = futures::mapValue(fv, [](int i){
        return makeFuture(i+1);
    });
```
```
    std::vector<Future<int>> fv;
    for (int i = 0; i < 10; i++){
        fv.emplace_back(makeFuture(i));
    }

    Future<double> f = folly::reduce(fv, 0.0, [](double a, int&& b){
        return a+b*10;
    });
```
总的来说，作为异步task的编排，continuation确实很方便，很直观的显示task的执行过程。但其实仔细思考我们就会发现，我们其实没有提及这些异步task或者future具体是如何被执行的，多个then操作，他们在同一个线程执行吗？还是不同的线程？这就要引入folly的promise/future框架一个很重要的东西：executor。

## Executor
在std中，promise/future的并没有过多的触及执行和调度的问题，虽然框架的成熟度是一个原因，但主要还是历史原因。c++委员会一直在致力于让promise/future更加的generic，整个任务图框架是一个漫长的演进过程。Executor提案P0443R14就是中间的一步。

Executor这个概念，主要是用于执行过程的抽象，对于任务图本身而言，我们为什么要关心如何执行调度呢？是多线程，还是内联，是使用CPU，还是GPU，这些其实都不是任务执行者关心的，对于consumer而言，他关心的是结果，是顺序，是我想做什么，而并非调度的细节。Executor将调度的细节隐藏了起来，对外提供了抽象统一的接口。

folly对executor提案给出了自己的实现方案。目前folly支持了下面几种executor(当然，他们都有统一的抽象基类):
1. InlineExecutor（默认）
2. （TBD)



# TBD





