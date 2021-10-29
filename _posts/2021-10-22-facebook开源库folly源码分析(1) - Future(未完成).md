---
layout: mypost
title: Facebook开源库folly源码分析(1) - Future
categories: [folly]
---
# 异步和并发
异步不一定并发，并发也不一定异步，这是一个老话题了，虽然异步和并发并不是一回事，但在实际应用的时候，并发和异步是经常在一起的，多线程中经常会有很多异步的操作。

c++11推出的future/promise模型就是一种异步编程框架。

在c++11以前，如果想要获取一些异步task的结果，通常需要依赖于callback函数，指针，或者类似的结果，对于异步这个行为的抽象是不够的的。C++11为了对异步编程提供更好的抽象，推出了future/promise模型。

本文不再讨论future/promise的设计理念，只集中在future/promise的源码实现上。

# 从std的future/promise谈起
std标准库的future/promise主要在`<future>`中。promise/future模型主要由下面几个部分组成：

1. futures类: future, shard_future
2. provider类: promise, packaged_task
3. provider函数: async

他们之间的关系可以简单描述为:
1. future是对异步任务的结果的封装, 可以通过future获取异步任务的结果。
2. promise则用来设置future里的数据，是数据的provider。

下面主要以gcc 8.4.1的实现来看std版本的实现。

## future和promise的简单使用示例
```
//创建一个promise对象
auto p = std::promise<int>();
auto f = promise.get_future();

//启动另外一个线程，并通过promise去写入数据
auto t = std::thread([&p]
    {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        p.set_value(1);
    });

// 获取数据，如果数据还没准备好就会阻塞
std::cout << f.get() << std::endl;
//future对象只能get()一次，不允许获取结果两次。
//std::cout << f.get() << std::endl; //throw exception
//promise对象只能set_value()一次，不允许set两次
//p.set_value(2); //throw exception
t.join();
```
future对象只支持移动，不支持拷贝，future的get()隐含有移动语义，所以只可以调用一次，对应的，promise对象也只能set_value一次。future和promise我们可以认为都是一次性的对象，使用完之后就不可以再去操作里面的数据了。这有点类似（仅仅是类似）管道或者go里面的channel。

实际上数据是存储在future对象里，future对象和promise对象共享了一个shared state对象的指针，通过这个state对象，promise可以通知到future那边，来判断继续处理业务或者错误。

网上有一张经典的描述future和promise本质的图。

![std_future_promise_1.jpg](std_future_promise_1.jpeg)


## future
std中的future其实是一个派生类，继承自__basic_future。shared_future也继承自__basic_future，我们后面会谈到。

`future`
```
template<typename _Res>
    class future : public __basic_future<_Res>
    {
        ...
      explicit
      future(const __state_type& __state) : _Base_type(__state) { }

    public:
        ...
      /// Retrieving the value
      _Res
      get()
      {
        typename _Base_type::_Reset __reset(*this);
        return std::move(this->_M_get_result()._M_value());
      }
        ...
    };
```
可以看到实际上每次get操作之后，都会reset. 这就是为什么不能get第二次的原因。

我们再看看父类，实际上__basic_future也是派生类，派生自__future_base。

`__basic_future`
```
/// Common implementation for future and shared_future.
template<typename _Res>
    class __basic_future : public __future_base
    {
    protected:
      typedef shared_ptr<_State_base>		__state_type;
      typedef __future_base::_Result<_Res>&	__result_type;

    private:
      __state_type 		_M_state;

        ...
    protected:
      /// Wait for the state to be ready and rethrow any stored exception
      __result_type
      _M_get_result() const
      {
        _State_base::_S_check(_M_state);
        _Result_base& __res = _M_state->wait();
        if (!(__res._M_error == 0))
          rethrow_exception(__res._M_error);
        return static_cast<__result_type>(__res);
      }
        ...
      struct _Reset
      {
        explicit _Reset(__basic_future& __fut) noexcept : _M_fut(__fut) { }
        ~_Reset() { _M_fut._M_state.reset(); }
        __basic_future& _M_fut;
      };
    };
```
__basic_future中维护了一个_State_base的shared_ptr _M_state，这个指针和promise那边共享一个_State_base。而真正的数据就在_State_base里面。比如，_M_get_result里面就通过_M_state->wait()去拿到真正的数据。_State_base的定义则还在基类__future_base里。

在每次获取数据的时候，都会通过_S_check检查_State_base的状态，如果有问题，会抛出future定义的一些异常。而reset就比较简单，直接将shared_ptr reset了。

`__future_base`
```
  /// Base class and enclosing scope.
  struct __future_base
  {
    /// Base class for results.
    struct _Result_base
    {
        ...
    };

    /// A unique_ptr for result objects.
    template<typename _Res>
      using _Ptr = unique_ptr<_Res, _Result_base::_Deleter>;

    /// A result object that has storage for an object of type _Res.
    template<typename _Res>
      struct _Result : _Result_base
      {
      private:
	__gnu_cxx::__aligned_buffer<_Res>	_M_storage;
	bool 					_M_initialized;

      public:
	typedef _Res result_type;
    ...
      };

    // Base class for various types of shared state created by an
    // asynchronous provider (such as a std::promise) and shared with one
    // or more associated futures.
    class _State_baseV2
    {
      typedef _Ptr<_Result_base> _Ptr_type;

      enum _Status : unsigned {
	__not_ready,
	__ready
      };

      _Ptr_type			_M_result;
      __atomic_futex_unsigned<>	_M_status;
      atomic_flag         	_M_retrieved = ATOMIC_FLAG_INIT;
      once_flag			_M_once;
    ...
    };
    ...
  };
```
现在就比较清晰了，__future_base定义了_Result结构用于存储数据，同时定义了_State_base (老版本的gcc, 比如4.8.2, 使用__State_base, 新版本的gcc则重新设计了__State_baseV2), __State_base用于封装Result的unique_ptr _M_result，同步用的锁_M_status，以及一些flag。

上文已经提到了，future获取数据实际是依靠_State_base的wait()去拿到真正的数据的。
```
     _Result_base&
      wait()
      {
	// Run any deferred function or join any asynchronous thread:
	_M_complete_async();
	// Acquire MO makes sure this synchronizes with the thread that made
	// the future ready.
	_M_status._M_load_when_equal(_Status::__ready, memory_order_acquire);
	return *_M_result;
      }
```
这里实际上是利用自旋锁和linux的futex系统调用来实现异步。我们就不展开分析了。

__State_base是一个比较关键的类，是provider和future交互的手段，也是整个promise/future模型最关键的部分。关于如何设置数据，我们下面讲到promise的数据会继续讨论。

### shared_future
如上文所述，future的get()函数是隐含着移动语义的，只可以调用一次。那如果我们想多次获取数据，或者想在多个线程里都获取数据，该怎么办呢？在这种情况下可以使用shared_future。

shared_future可以从future对象隐式转换，也可以通过显式地调用future的share()转换，无论哪一种原来的future对象都将失效。

shared_future对象的get()函数是可以多次调用的。从源码上可以清晰地看到区别。

`shared_future`
```
/// Primary template for shared_future.
  template<typename _Res>
    class shared_future : public __basic_future<_Res>
    {
        ...
      /// Retrieving the value
      const _Res&
      get() const { return this->_M_get_result()._M_value(); }
    };
```

shard_future使用的示例
```
	std::promise<int> p;
	shared_future<int> sf = p.get_future();

	p.set_value(1);
	cout<<"sf="<<sf.get()<<endl;
	cout<<"sf="<<sf.get()<<endl;
```

## promise
promise的定义相比future而言，简单了很多。

`promise`
```
 /// Primary template for promise
  template<typename _Res>
    class promise
    {
      typedef __future_base::_State_base 	_State;
      typedef __future_base::_Result<_Res>	_Res_type;
      typedef __future_base::_Ptr<_Res_type>	_Ptr_type;
      template<typename, typename> friend class _State::_Setter;
      friend _State;

      shared_ptr<_State>                        _M_future;
      _Ptr_type                                 _M_storage;

    public:
      promise()
      : _M_future(std::make_shared<_State>()),
	_M_storage(new _Res_type())
      { }
      ...

      // Retrieving the result
      future<_Res>
      get_future()
      { return future<_Res>(_M_future); }

      // Setting the result
      void
      set_value(const _Res& __r)
      { _M_future->_M_set_result(_State::__setter(this, __r)); }
      ...

    };
```
可以看到promise里定义了_State_base的shared_ptr _M_future, 顾名思义，这个state对象会用于get_future的时候创建future。同时还定义了一个存储_Res_type对象的unique_ptr，用于未来设置数据，最终传给future。

每当promise通过set_value设置数据的时候，会调用_State_base类提供的_M_set_result去给共享的_M_future设置数据。_M_set_result一方面设置数据，另一方面会通知其他的异步任务数据已经准备好了。
```
      void
      _M_set_result(function<_Ptr_type()> __res, bool __ignore_failure = false)
      {
	bool __did_set = false;
        // all calls to this function are serialized,
        // side-effects of invoking __res only happen once
	call_once(_M_once, &_State_baseV2::_M_do_set, this,
		  std::__addressof(__res), std::__addressof(__did_set));
	if (__did_set)
	  // Use release MO to synchronize with observers of the ready state.
	  _M_status._M_store_notify_all(_Status::__ready,
					memory_order_release);
	else if (!__ignore_failure)
          __throw_future_error(int(future_errc::promise_already_satisfied));
      }
```
实际上数据的设置是通过_State_base的_M_do_set函数，准确的说，是通过_State::__setter来设置的, 最终数据会swap到state对象_M_future的_M_result成员里。
```
     _M_do_set(function<_Ptr_type()>* __f, bool* __did_set)
      {
        _Ptr_type __res = (*__f)();
        // Notify the caller that we did try to set; if we do not throw an
        // exception, the caller will be aware that it did set (e.g., see
        // _M_set_result).
	*__did_set = true;
        _M_result.swap(__res); // nothrow
      }
```
setter本身就简单很多了, 实际上利用仿函数_Setter来处理有数据的场景和error的场景。首先数据被拷贝构造放到了promise的_M_storage里面，_M_storage上面提到过，是存储数据类型的unique_ptr。然后对象的所有权再被转移出去, 放到state对象里。
```
     // set lvalues
      template<typename _Res, typename _Arg>
        struct _Setter<_Res, _Arg&>
        {
            ...

	  // Used by std::promise to copy construct the result.
          typename promise<_Res>::_Ptr_type operator()() const
          {
            _M_promise->_M_storage->_M_set(*_M_arg);
            return std::move(_M_promise->_M_storage);
          }
          promise<_Res>*    _M_promise;
          _Arg*             _M_arg;
        };
        ...

      template<typename _Res, typename _Arg>
        static _Setter<_Res, _Arg&&>
        __setter(promise<_Res>* __prom, _Arg&& __arg)
        {
	  _S_check(__prom->_M_future);
          return _Setter<_Res, _Arg&&>{ __prom, std::__addressof(__arg) };
        }
```
于是乎数据就是这么一层层的调用，最终被放到了promise和future对象共享的_State_base，于是future就可以异步获取数据了。

## packaged_task
(TBD)





