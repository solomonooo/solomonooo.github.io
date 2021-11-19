---
layout: mypost
title: Facebook开源库folly源码分析(1) - Future
categories: [folly]
---
# 异步和并发
异步不一定并发，并发也不一定异步，这是一个老话题了，虽然异步和并发并不是一回事，但在实际应用的时候，并发和异步是经常在一起的，多线程中经常会有很多异步的操作。

c++11推出的`future/promise`模型就是一种异步编程框架。

在c++11以前，如果想要获取一些异步task的结果，通常需要依赖于callback函数，指针，或者类似的结果，对于异步这个行为的抽象是不够的的。C++11为了对异步编程提供更好的抽象，推出了`future/promise`模型。

本文不再讨论`future/promise`的设计理念，只集中在`future/promise`的源码实现上。

# 从std的future/promise谈起
std标准库的`future/promise`主要在`<future>`中。`promise/future`模型主要由下面几个部分组成：

1. futures类: `future`, `shard_future`
2. provider类: `promise`, `packaged_task`
3. provider函数: `async`

他们之间的关系可以简单描述为:
1. `future`是对异步任务的结果的封装, 可以通过`future`获取异步任务的结果。
2. `promise`则用来设置`future`里的数据，是数据的provider。

下面主要以`gcc 8.4.1`的实现来看`std`版本的实现。

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
`future`对象只支持移动，不支持拷贝，`future`的`get()`隐含有移动语义，所以只可以调用一次，对应的，`promise`对象也只能`set_value`一次。`future`和`promise`我们可以认为都是一次性的对象，使用完之后就不可以再去操作里面的数据了。这有点类似（仅仅是类似）管道或者go里面的channel。

实际上数据是存储在一个共享state对象里面，`future`对象和`promise`对象共享这个shared state对象的指针，通过这个state对象，`promise`可以通知到`future`那边，来判断继续处理业务或者错误。

网上有一张经典的描述`future`和`promise`本质的图。

![std_future_promise_1.jpg](std_future_promise_1.jpeg)

## future
`std`中的`future`其实是一个派生类，继承自`__basic_future`。`shared_future`也继承自`__basic_future`，我们后面会谈到。`

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

我们再看看父类，实际上`__basic_future`也是派生类，派生自`__future_base`。

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
`__basic_future`中维护了一个`_State_base`的shared_ptr `_M_state`，这个指针和`promise`那边共享一个`_State_base`。而真正的数据就在`_State_base`里面。比如，`_M_get_result`里面就通过`_M_state->wait()`去拿到真正的数据。`_State_base`的定义则还在基类`__future_base`里。

在每次获取数据的时候，都会通过`_S_check`检查`_State_base`的状态，如果有问题，会抛出`future`定义的一些异常。而reset就比较简单，直接将shared_ptr reset了。

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
现在就比较清晰了，`__future_base`定义了`_Result`结构用于存储数据，同时定义了`_State_base` (老版本的gcc, 比如4.8.2, 使用`_State_base`, 新版本的gcc则重新设计了`_State_baseV2`), `_State_base`用于封装`Result`的unique_ptr `_M_result`，同步用的锁`_M_status`，以及一些flag。

上文已经提到了，`future`获取数据实际是依靠`_State_base`的`wait()`去拿到真正的数据的。
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
_M_complete_async是一个虚函数，对于promise的场景下，其实没有任何行为。他是为了其他派生的shared state使用的。比如后面会提到的async, async使用的派生的shared state对象就自己实现了_M_complete_async, 用于等待其他线程，或者延迟执行函数。

_M_load_when_equal用于和provider之间同步，当provider侧通知到future这边，就可以从_M_result获取数据了。这里实际上是利用自旋锁和linux的futex系统调用来实现异步。我们就不展开分析了。

`__State_base`是一个比较关键的类，是provider和future交互的手段，也是整个`promise/future`模型最关键的部分。关于如何设置数据，我们下面讲到promise的数据会继续讨论。

### shared_future
如上文所述，`future`的`get()`函数是隐含着移动语义的，只可以调用一次。那如果我们想多次获取数据，或者想在多个线程里都获取数据，该怎么办呢？在这种情况下可以使用`shared_future`。

`shared_future`可以从`future`对象隐式转换，也可以通过显式地调用`future`的`share()`转换，无论哪一种原来的`future`对象都将失效。

`shared_future`对象的`get()`函数是可以多次调用的。从源码上可以清晰地看到区别。

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
`promise`的定义相比`future`而言，简单了很多。

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
可以看到`promise`里定义了`_State_base`的shared_ptr `_M_future`, 顾名思义，这个state对象会用于get_future的时候创建`future`。同时还定义了一个存储`_Res_type`对象的unique_ptr，用于未来设置数据，最终传给`future`。

每当`promise`通过set_value设置数据的时候，会调用`_State_base`类提供的`_M_set_result`去给共享的`_M_future`设置数据。`_M_set_result`一方面设置数据，另一方面会通知其他的异步任务数据已经准备好了。
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
实际上数据的设置是通过`_State_base`的`_M_do_set`函数，准确的说，是通过`_State::__setter`来设置的, 最终数据会swap到state对象`_M_future`的`_M_result`成员里。
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
setter本身就简单很多了, 实际上利用仿函数`_Setter`来处理有数据的场景和error的场景。首先数据被拷贝构造放到了`promise`的`_M_storage`里面，`_M_storage`上面提到过，是存储数据类型的unique_ptr。然后对象的所有权再被转移出去, 放到state对象里。
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
于是乎数据就是这么一层层的调用，最终被放到了`promise`和`future`对象共享的`_State_base`，于是`future`就可以异步获取数据了。

## packaged_task
`packaged_task`和`promise`一样，都是异步provider。`promise`是对异步任务结果的封装，而`packaged_task`则是对可调用对象(lamda, function等等)的异步封装，更进一步方便实现异步操作，可以通过`future`快速获取可调用对象的返回值。

举个例子。
```
    std::packaged_task<int(int, int)> pt([](int a, int b) {
                return a+b;
            });
    auto f = pt.get_future();

    pt(1, 2);
    cout<<"f="<<f.get()<<endl;
```
`packaged_task`的实现和`promise`很类似，`packaged_task`对象里也维护了一个shared_ptr, 区别只是promise维护的是`_State_base`，而packaged_task维护的是他的派生类`_Task_state_base`。

`_Task_state_base`作为`_State_base`的派生类，不仅仅维护了作为shared state必须的数据和锁，还保存了可调用对象(实际上还在`_Task_state_base`的派生类里)。

通过简单的重载操作符(), 就可以方便的执行可调用对象/函数了。

`packaged_task`也和`promise`一样，只支持move，不支持copy。
```
/// packaged_task
  template<typename _Res, typename... _ArgTypes>
    class packaged_task<_Res(_ArgTypes...)>
    {
      typedef __future_base::_Task_state_base<_Res(_ArgTypes...)> _State_type;
      shared_ptr<_State_type>                   _M_state;

    public:
      ...

      template<typename _Fn, typename = typename
	       __constrain_pkgdtask<packaged_task, _Fn>::__type>
	explicit
	packaged_task(_Fn&& __fn)
	: packaged_task(allocator_arg, std::allocator<int>(),
			std::forward<_Fn>(__fn))
	{ }

      // _GLIBCXX_RESOLVE_LIB_DEFECTS
      // 2097.  packaged_task constructors should be constrained
      // 2407. [this constructor should not be] explicit
      template<typename _Fn, typename _Alloc, typename = typename
	       __constrain_pkgdtask<packaged_task, _Fn>::__type>
	packaged_task(allocator_arg_t, const _Alloc& __a, _Fn&& __fn)
	: _M_state(__create_task_state<_Res(_ArgTypes...)>(
		    std::forward<_Fn>(__fn), __a))
	{ }

        ...
      // Execution
      void
      operator()(_ArgTypes... __args)
      {
	__future_base::_State_base::_S_check(_M_state);
	_M_state->_M_run(std::forward<_ArgTypes>(__args)...);
      }

        ...

      // Result retrieval
      future<_Res>
      get_future()
      { return future<_Res>(_M_state); }
        ...

    };
```
`_Task_state_base`作为task state的基类，只简单定义了存储数据的unique_ptr指针`_M_result`，和其他必须的虚函数接口，实现都交给子类了。

```
 // Holds storage for a packaged_task's result.
  template<typename _Res, typename... _Args>
    struct __future_base::_Task_state_base<_Res(_Args...)>
    : __future_base::_State_base
    {
      typedef _Res _Res_type;
        ...

      typedef __future_base::_Ptr<_Result<_Res>> _Ptr_type;
      _Ptr_type _M_result;
    };
```
这里我们就可以看到`_Task_state_base`有一个`_M_result`，父类`_State_base`里也有一个`_M_result`，这里是不是冗余了？

其实并不是这样。在这里的两个`_M_result`其实是为了设置shared state数据的接口的统一。回想一下`promise`是如何设置数据的。`promise`在set value的时候，数据先被拷贝构造放到了promise的`_M_storage`里面，然后才将对象的所有权转移出去, 放到state对象里。这里对task state也是一样的操作。

可调用对象的结果被临时放到了task state的`_M_result`里，然后再被转移到父类的`_M_result`里。这样就保持了数据读取的接口统一。task state为了实现这一目标，通过`_S_task_setter`来封装task的返回值和调用函数。所以对于future而言，无论数据是来自哪种provider，访问的接口都是一样的。

而可调用对象或者说task，其实是存在`_Task_state_base`的派生类`_Task_state`里面的。`_M_impl`成员保存了可调用对象。
```
// Holds a packaged_task's stored task.
  template<typename _Fn, typename _Alloc, typename _Res, typename... _Args>
    struct __future_base::_Task_state<_Fn, _Alloc, _Res(_Args...)> final
    : __future_base::_Task_state_base<_Res(_Args...)>
    {
        ...

      struct _Impl : _Alloc
      {
	template<typename _Fn2>
	  _Impl(_Fn2&& __fn, const _Alloc& __a)
	  : _Alloc(__a), _M_fn(std::forward<_Fn2>(__fn)) { }
	_Fn _M_fn;
      } _M_impl;
    };
```

## async
`async`是c++提供的provider函数，他的功能很类似`packaged_task`，也是对可调用对象的封装，`async`会其直接启动异步任务，然后返回`future`，连get_future都替我们省掉了。

举个例子。
```
    auto future = std::async([](int a, int b) {
				return a+b;
			}, 1, 2);

    std::cout << future.get() << std::endl;
```
`async`有点类似`std::thread`。实际上如果参数是`launch::async`的情况下，`async`内部也确实启动了一个新的线程(就是直接用`std::thread`)去执行异步任务。


`async`的第一个参数是`std::launch`枚举值，他提供了2种`async`的启动方式: `launch::async`和`launch::deferred`。`launch::async`我们上面提过了，`async`会启动一个新的线程去执行异步任务，并返回future。而`launch::deferred`比较特殊一点，其实`async`内部并不会启动新线程去执行，而且顾名思义，将调用函数延迟执行，在获取数据的那一刻才执行可调用对象本身。

`async`的默认参数是`launch::async|launch::deferred`, 具体的行为由系统根据决定。在系统资源足够的情况下，`async`会走`launch::async`的模式创建新的线程，而当资源不足的时候，`async`会以延迟调用的方式执行。

下面我们来看看async的实现。
```
 /// async
  template<typename _Fn, typename... _Args>
    future<__async_result_of<_Fn, _Args...>>
    async(launch __policy, _Fn&& __fn, _Args&&... __args)
    {
      std::shared_ptr<__future_base::_State_base> __state;
      if ((__policy & launch::async) == launch::async)
	{
	  __try
	    {
	      __state = __future_base::_S_make_async_state(
		  std::thread::__make_invoker(std::forward<_Fn>(__fn),
					      std::forward<_Args>(__args)...)
		  );
	    }
#if __cpp_exceptions
	  catch(const system_error& __e)
	    {
	      if (__e.code() != errc::resource_unavailable_try_again
		  || (__policy & launch::deferred) != launch::deferred)
		throw;
	    }
#endif
	}
      if (!__state)
	{
	  __state = __future_base::_S_make_deferred_state(
	      std::thread::__make_invoker(std::forward<_Fn>(__fn),
					  std::forward<_Args>(__args)...));
	}
      return future<__async_result_of<_Fn, _Args...>>(__state);
    }

  /// async, potential overload
  template<typename _Fn, typename... _Args>
    inline future<__async_result_of<_Fn, _Args...>>
    async(_Fn&& __fn, _Args&&... __args)
    {
      return std::async(launch::async|launch::deferred,
			std::forward<_Fn>(__fn),
			std::forward<_Args>(__args)...);
    }
```
可以很直观的看到。其实对于不同的启动模式，`async`创建了不同share state。对于async模式，创建`_Async_state_impl`, 对于deferred模式，创建`_Deferred_state`。他们都派生自`_State_base`，按需去添加自己的私有成员。

创建share state使用的辅助函数。
```
  template<typename _BoundFn>
    inline std::shared_ptr<__future_base::_State_base>
    __future_base::_S_make_deferred_state(_BoundFn&& __fn)
    {
      typedef typename remove_reference<_BoundFn>::type __fn_type;
      typedef _Deferred_state<__fn_type> __state_type;
      return std::make_shared<__state_type>(std::move(__fn));
    }

  template<typename _BoundFn>
    inline std::shared_ptr<__future_base::_State_base>
    __future_base::_S_make_async_state(_BoundFn&& __fn)
    {
      typedef typename remove_reference<_BoundFn>::type __fn_type;
      typedef _Async_state_impl<__fn_type> __state_type;
      return std::make_shared<__state_type>(std::move(__fn));
    }
```

`_Async_state_impl`对象在创建时启动线程去运行可调用对象，析构时join等待线程执行结束。其中_M_thread是起父类`_Async_state_commonV2`定义的成员，而`_Async_state_commonV2`则派生自`_State_base`，对线程相关的操作进行了封装。
```
  // Shared state created by std::async().
  // Starts a new thread that runs a function and makes the shared state ready.
  template<typename _BoundFn, typename _Res>
    class __future_base::_Async_state_impl final
    : public __future_base::_Async_state_commonV2
    {
    public:
      explicit
      _Async_state_impl(_BoundFn&& __fn)
      : _M_result(new _Result<_Res>()), _M_fn(std::move(__fn))
      {
	_M_thread = std::thread{ [this] {
	    __try
	      {
		_M_set_result(_S_task_setter(_M_result, _M_fn));
	      }
	    __catch (const __cxxabiv1::__forced_unwind&)
	      {
              ...
	      }
        } };
      }
        ...
      
      // thread can be referring to this state if it is being destroyed.
      ~_Async_state_impl() { if (_M_thread.joinable()) _M_thread.join(); }

    private:
      typedef __future_base::_Ptr<_Result<_Res>> _Ptr_type;
      _Ptr_type _M_result;
      _BoundFn _M_fn;
    };
```
`_Deferred_state`对象在创建的时候直接维护可调用对象，然后等待调用触发。触发的线程可以是调用`async`的线程，也可以是其他的线程。完全取决于我们如何使用async返回的future。但无论如何，可调用对象只会call一次。
```
// Shared state created by std::async().
  // Holds a deferred function and storage for its result.
  template<typename _BoundFn, typename _Res>
    class __future_base::_Deferred_state final
    : public __future_base::_State_base
    {
    public:
      explicit
      _Deferred_state(_BoundFn&& __fn)
      : _M_result(new _Result<_Res>()), _M_fn(std::move(__fn))
      { }

    private:
      typedef __future_base::_Ptr<_Result<_Res>> _Ptr_type;
      _Ptr_type _M_result;
      _BoundFn _M_fn;

      // Run the deferred function.
      virtual void
      _M_complete_async()
      {
	// Multiple threads can call a waiting function on the future and
	// reach this point at the same time. The call_once in _M_set_result
	// ensures only the first one run the deferred function, stores the
	// result in _M_result, swaps that with the base _M_result and makes
	// the state ready. Tell _M_set_result to ignore failure so all later
	// calls do nothing.
        _M_set_result(_S_task_setter(_M_result, _M_fn), true);
      }
        ...

    };
```

## shared state
其实我们上面已经很多次提到shared state了。`_State_base`是一个非常关键的对象，我们再来总结一下他和他的派生类：
1. `_State_base`/`_State_baseV2` : 基类，promise使用的对象。
2. `_Task_state_base`/`_Task_state` : packaged_task使用的派生对象。
3. `_Async_state_impl`/`_Async_state_commonV2`/`_Deferred_state` : async使用的派生对象。

## 总结
到此为止，我们分析完了`std`标准库对于`future/promise`模型的实现，本质上来说这是一个生产者/消费者的管道式模型。

我们可以看到，整个异步任务模型是完全依赖`_State_base`这个共享对象来实现的。对于不同的`provider`，`std`分别派生不同的state对象，去实现自己定制化的封装。 

`provider`负责创建state对象，然后在必要的时候利用state对象去构建`future`。

`future`负责消费state对象，一般来说是一次性消费，当然也可以实现多次消费。

下篇我们来看看`folly`的实现，感受一下facebook是如何更进一步去实现`future/promise`模型的。




