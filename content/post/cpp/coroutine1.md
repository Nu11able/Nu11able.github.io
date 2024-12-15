---
title: c++协程原理
date: 2024-12-11
tags:
 - cpp
 - coroutine
categories:
 - cpp
---
# c++协程原理
本文会先后介绍协程的关键字、使用方法以及它的原理，文章最后会给出一个例子来帮助加深理解。

## Q:协程被定义为“可中断”的函数，编译器是如何支持“可中断”这个操作的?
先提出本文最重要一个问题，答案会在后续的一个个子问题中得到解答，我认为解答了这个问题也就理解了协程的原理。

### Q:c++里面什么样的函数算是一个协程?
函数中含有以下关键字
- co_await
- co_yield
- co_return
其中 co_yield和co_return都是用来返回值的，co_await是用来等待异步操作的(另一个协程)。
```cpp
int example() {
    co_yield 1;
    co_yield 2;
    co_yield 3;
    co_return 4;
}
int main() {
    auto ret = example();
    // 此时ret是一个int类型的值，值为1  我如何恢复example的执行?
}
```
实际上诉的代码并不能够通过编译，一个协程函数的返回类型应该是一个`协程类型`，而不是一个普通的类型。

### Q:协程函数的返回类型是什么?
在此需要先提出三个概念
- 协程promise(the promise object)
    > manipulated from inside the coroutine. The coroutine submits its result or exception through this object. Promise objects are in no way related to std::promise.

    promise是一个对象，但是它和std::promise没有任何关系。
- 协程帧(coroutine state)
    协程帧是由编译器生成(一个由编译器生成的类型)，它(的成员变量)包含了协程的状态信息，比如协程的参数、当前执行位置，协程的promise对象等。
- 协程句柄(coroutine handle)
    > manipulated from outside the coroutine. This is a non-owning handle used to resume execution of the coroutine or to destroy the coroutine frame.

    句柄是一个对象，它用来恢复协程的执行或者销毁协程帧。它内部拥有一个指向协程帧的指针。

协程函数的返回类型、协程的执行结果都是由promise对象提供。在此先给出一部分promise的定义
```cpp
struct promise_type {
    return_type get_return_object(); // return_type即为协程函数的返回类型
};
```
return_type和promise_type都由用户自行定义，但是promise_type必须包含一个get_return_object方法，这个方法返回一个return_type类型的对象。且return_type必须含有一个promise_type类型。
```cpp
struct return_type;
struct promise_type {
    return_type get_return_object();
};
struct return_type {
    using promise_type = ::promise_type;
}

// 当然你也可以这样定义
struct return_type {
    struct promise_type {
        return_type get_return_object();
    };
}
```
由此我们得到了协程函数的返回类型，但还远远不够。再继续下去之前我们需要先了解编译器协程的执行流程。
```cpp
// 协程帧结构(由编译器生成)
struct coroutine_state {
    promise_type promise;
    // other members
    // ...
};
```

- 1. 使用`operator new`分配一个协程帧的内存
- 2. 将所有参数拷贝到协程帧
- 3. 调用promise_type的构造函数
- 4. 调用promise_type的get_return_object方法，将结果并保存为本地变量，以便后续返回
- 5. 调用promise_type的initial_suspend方法并等待它的结果(`co_await promise.initial_suspend()`)
- 6. 当`co_await promise.initial_suspend()`恢复之后开始执行协程函数


## 例子
### 这是一个协程的[例子](https://devdocs.io/cpp/language/coroutines)
```cpp
#include <coroutine>
#include <cstdint>
#include <exception>
#include <iostream>
 
template<typename T>
struct Generator
{
    // The class name 'Generator' is our choice and it is not required for coroutine
    // magic. Compiler recognizes coroutine by the presence of 'co_yield' keyword.
    // You can use name 'MyGenerator' (or any other name) instead as long as you include
    // nested struct promise_type with 'MyGenerator get_return_object()' method.
 
    struct promise_type;
    using handle_type = std::coroutine_handle<promise_type>;
 
    struct promise_type // required
    {
        T value_;
        std::exception_ptr exception_;
 
        Generator get_return_object()
        {
            return Generator(handle_type::from_promise(*this));
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void unhandled_exception() { exception_ = std::current_exception(); } // saving
                                                                              // exception
 
        template<std::convertible_to<T> From> // C++20 concept
        std::suspend_always yield_value(From&& from)
        {
            value_ = std::forward<From>(from); // caching the result in promise
            return {};
        }
        void return_void() {}
    };
 
    handle_type h_;
 
    Generator(handle_type h) : h_(h) {}
    ~Generator() { h_.destroy(); }
    explicit operator bool()
    {
        fill(); // The only way to reliably find out whether or not we finished coroutine,
                // whether or not there is going to be a next value generated (co_yield)
                // in coroutine via C++ getter (operator () below) is to execute/resume
                // coroutine until the next co_yield point (or let it fall off end).
                // Then we store/cache result in promise to allow getter (operator() below
                // to grab it without executing coroutine).
        return !h_.done();
    }
    T operator()()
    {
        fill();
        full_ = false; // we are going to move out previously cached
                       // result to make promise empty again
        return std::move(h_.promise().value_);
    }
 
private:
    bool full_ = false;
 
    void fill()
    {
        if (!full_)
        {
            h_();
            if (h_.promise().exception_)
                std::rethrow_exception(h_.promise().exception_);
            // propagate coroutine exception in called context
 
            full_ = true;
        }
    }
};
 
Generator<std::uint64_t>
fibonacci_sequence(unsigned n)
{
    if (n == 0)
        co_return;
 
    if (n > 94)
        throw std::runtime_error("Too big Fibonacci sequence. Elements would overflow.");
 
    co_yield 0;
 
    if (n == 1)
        co_return;
 
    co_yield 1;
 
    if (n == 2)
        co_return;
 
    std::uint64_t a = 0;
    std::uint64_t b = 1;
 
    for (unsigned i = 2; i < n; ++i)
    {
        std::uint64_t s = a + b;
        co_yield s;
        a = b;
        b = s;
    }
}
 
int main()
{
    try
    {
        auto gen = fibonacci_sequence(10); // max 94 before uint64_t overflows
 
        for (int j = 0; gen; ++j)
            std::cout << "fib(" << j << ")=" << gen() << '\n';
    }
    catch (const std::exception& ex)
    {
        std::cerr << "Exception: " << ex.what() << '\n';
    }
    catch (...)
    {
        std::cerr << "Unknown exception.\n";
    }
}
```

### 经过[cppinsights](https://cppinsights.io/)转化后
```cpp
/*************************************************************************************
 * NOTE: The coroutine transformation you've enabled is a hand coded transformation! *
 *       Most of it is _not_ present in the AST. What you see is an approximation.   *
 *************************************************************************************/
#include <coroutine>
#include <cstdint>
#include <exception>
#include <iostream>

template<typename T>
struct Generator
{
  struct promise_type;
  using handle_type = std::coroutine_handle<promise_type>;
  struct promise_type
  {
    T value_;
    std::exception_ptr exception_;
    inline Generator<T> get_return_object()
    {
      return Generator<T>(handle_type::from_promise(*this));
    }
    
    inline std::suspend_always initial_suspend()
    {
      return {};
    }
    
    inline std::suspend_always final_suspend() noexcept
    {
      return {};
    }
    
    inline void unhandled_exception()
    {
      this->exception_.operator=(std::current_exception());
    }
    
    template<std::convertible_to<T> From>
    inline std::suspend_always yield_value(From && from)
    {
      this->value_ = std::forward<From>(from);
      return {};
    }
    inline void return_void()
    {
    }
    
  };
  
  handle_type h_;
  inline Generator(handle_type h)
  : h_(h)
  {
  }
  
  inline ~Generator()
  {
    this->h_.destroy();
  }
  
  inline explicit operator bool ()
  {
    this->fill();
    return !this->h_.done();
  }
  
  inline T operator()()
  {
    this->fill();
    this->full_ = false;
    return std::move(this->h_.promise().value_);
  }
  
  
  private: 
  bool full_;
  inline void fill()
  {
    if(!this->full_) {
      this->h_();
      if(this->h_.promise().exception_) {
        std::rethrow_exception(this->h_.promise().exception_);
      } 
      
      this->full_ = true;
    } 
    
  }
  
};

/* First instantiated from: insights.cpp:80 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
struct Generator<unsigned long>
{
  struct promise_type
  {
    unsigned long value_;
    std::exception_ptr exception_;
    inline Generator<unsigned long> get_return_object()
    {
      return Generator<unsigned long>(Generator<unsigned long>(std::coroutine_handle<promise_type>::from_promise(*this)));
    }
    
    inline std::suspend_always initial_suspend()
    {
      return {};
    }
    
    inline std::suspend_always final_suspend() noexcept
    {
      return {};
    }
    
    inline void unhandled_exception()
    {
      this->exception_.operator=(std::current_exception());
    }
    
    template<std::convertible_to<T> From>
    inline std::suspend_always yield_value(From && from);
    
    /* First instantiated from: insights.cpp:88 */
    #ifdef INSIGHTS_USE_TEMPLATE
    template<>
    inline std::suspend_always yield_value<int>(int && from)
    {
      this->value_ = static_cast<unsigned long>(std::forward<int>(from));
      return {};
    }
    #endif
    
    
    /* First instantiated from: insights.cpp:104 */
    #ifdef INSIGHTS_USE_TEMPLATE
    template<>
    inline std::suspend_always yield_value<unsigned long &>(unsigned long & from)
    {
      this->value_ = std::forward<unsigned long &>(from);
      return {};
    }
    #endif
    
    inline void return_void()
    {
    }
    
    // inline ~promise_type() noexcept = default;
  };
  
  using handle_type = std::coroutine_handle<promise_type>;
  struct promise_type;
  std::coroutine_handle<promise_type> h_;
  inline Generator(std::coroutine_handle<promise_type> h)
  : h_{std::coroutine_handle<promise_type>(h)}
  , full_{false}
  {
  }
  
  inline ~Generator() noexcept
  {
    this->h_.destroy();
  }
  
  inline explicit operator bool ()
  {
    this->fill();
    return !this->h_.done();
  }
  
  inline unsigned long operator()()
  {
    this->fill();
    this->full_ = false;
    return std::move(this->h_.promise().value_);
  }
  
  
  private: 
  bool full_;
  inline void fill()
  {
    if(!this->full_) {
      this->h_.operator()();
      if(this->h_.promise().exception_.operator bool()) {
        std::rethrow_exception(std::__exception_ptr::exception_ptr(this->h_.promise().exception_));
      } 
      
      this->full_ = true;
    } 
    
  }
  
  public: 
};

#endif

struct __fibonacci_sequenceFrame
{
  void (*resume_fn)(__fibonacci_sequenceFrame *);
  void (*destroy_fn)(__fibonacci_sequenceFrame *);
  std::__coroutine_traits_impl<Generator<unsigned long> >::promise_type __promise;
  int __suspend_index;
  bool __initial_await_suspend_called;
  unsigned int n;
  std::uint64_t a;
  std::uint64_t b;
  unsigned int i;
  std::uint64_t s;
  std::suspend_always __suspend_80_1;
  std::suspend_always __suspend_88_5;
  std::suspend_always __suspend_93_5;
  std::suspend_always __suspend_104_9;
  std::suspend_always __suspend_80_1_1;
};

Generator<unsigned long> fibonacci_sequence(unsigned int n)
{
  /* Allocate the frame including the promise */
  /* Note: The actual parameter new is __builtin_coro_size */
  __fibonacci_sequenceFrame * __f = reinterpret_cast<__fibonacci_sequenceFrame *>(operator new(sizeof(__fibonacci_sequenceFrame)));
  __f->__suspend_index = 0;
  __f->__initial_await_suspend_called = false;
  __f->n = std::forward<unsigned int>(n);
  
  /* Construct the promise. */
  new (&__f->__promise)std::__coroutine_traits_impl<Generator<unsigned long> >::promise_type{};
  
  /* Forward declare the resume and destroy function. */
  void __fibonacci_sequenceResume(__fibonacci_sequenceFrame * __f);
  void __fibonacci_sequenceDestroy(__fibonacci_sequenceFrame * __f);
  
  /* Assign the resume and destroy function pointers. */
  __f->resume_fn = &__fibonacci_sequenceResume;
  __f->destroy_fn = &__fibonacci_sequenceDestroy;
  
  /* Call the made up function with the coroutine body for initial suspend.
     This function will be called subsequently by coroutine_handle<>::resume()
     which calls __builtin_coro_resume(__handle_) */
  __fibonacci_sequenceResume(__f);
  
  
  return __f->__promise.get_return_object();
}

/* This function invoked by coroutine_handle<>::resume() */
void __fibonacci_sequenceResume(__fibonacci_sequenceFrame * __f)
{
  try 
  {
    /* Create a switch to get to the correct resume point */
    switch(__f->__suspend_index) {
      case 0: break;
      case 1: goto __resume_fibonacci_sequence_1;
      case 2: goto __resume_fibonacci_sequence_2;
      case 3: goto __resume_fibonacci_sequence_3;
      case 4: goto __resume_fibonacci_sequence_4;
    }
    
    /* co_await insights.cpp:80 */
    __f->__suspend_80_1 = __f->__promise.initial_suspend();
    if(!__f->__suspend_80_1.await_ready()) {
      __f->__suspend_80_1.await_suspend(std::coroutine_handle<Generator<unsigned long>::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
      __f->__suspend_index = 1;
      __f->__initial_await_suspend_called = true;
      return;
    } 
    
    __resume_fibonacci_sequence_1:
    __f->__suspend_80_1.await_resume();
    if(__f->n == 0) {
      /* co_return insights.cpp:83 */
      __f->__promise.return_void();
    } 
    
    if(__f->n > 94) {
      throw std::runtime_error(std::runtime_error("Too big Fibonacci sequence. Elements would overflow."));
    } 
    
    
    /* co_yield insights.cpp:88 */
    __f->__suspend_88_5 = __f->__promise.yield_value<int>(0);
    if(!__f->__suspend_88_5.await_ready()) {
      __f->__suspend_88_5.await_suspend(std::coroutine_handle<Generator<unsigned long>::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
      __f->__suspend_index = 2;
      return;
    } 
    
    __resume_fibonacci_sequence_2:
    __f->__suspend_88_5.await_resume();
    if(__f->n == 1) {
      /* co_return insights.cpp:91 */
      __f->__promise.return_void();
    } 
    
    
    /* co_yield insights.cpp:93 */
    __f->__suspend_93_5 = __f->__promise.yield_value<int>(1);
    if(!__f->__suspend_93_5.await_ready()) {
      __f->__suspend_93_5.await_suspend(std::coroutine_handle<Generator<unsigned long>::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
      __f->__suspend_index = 3;
      return;
    } 
    
    __resume_fibonacci_sequence_3:
    __f->__suspend_93_5.await_resume();
    if(__f->n == 2) {
      /* co_return insights.cpp:96 */
      __f->__promise.return_void();
    } 
    
    __f->a = 0;
    __f->b = 1;
    for(__f->i = 2; __f->i < __f->n; ++__f->i) {
      __f->s = (__f->a + __f->b);
      
      /* co_yield insights.cpp:104 */
      __f->__suspend_104_9 = __f->__promise.yield_value<unsigned long &>(__f->s);
      if(!__f->__suspend_104_9.await_ready()) {
        __f->__suspend_104_9.await_suspend(std::coroutine_handle<Generator<unsigned long>::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
        __f->__suspend_index = 4;
        return;
      } 
      
      __resume_fibonacci_sequence_4:
      __f->__suspend_104_9.await_resume();
      __f->a = __f->b;
      __f->b = __f->s;
    }
    
    goto __final_suspend;
  } catch(...) {
    if(!__f->__initial_await_suspend_called) {
      throw ;
    } 
    
    __f->__promise.unhandled_exception();
  }
  
  __final_suspend:
  
  /* co_await insights.cpp:80 */
  __f->__suspend_80_1_1 = __f->__promise.final_suspend();
  if(!__f->__suspend_80_1_1.await_ready()) {
    __f->__suspend_80_1_1.await_suspend(std::coroutine_handle<Generator<unsigned long>::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
    return;
  } 
  
  __f->destroy_fn(__f);
}

/* This function invoked by coroutine_handle<>::destroy() */
void __fibonacci_sequenceDestroy(__fibonacci_sequenceFrame * __f)
{
  /* destroy all variables with dtors */
  __f->~__fibonacci_sequenceFrame();
  /* Deallocating the coroutine frame */
  /* Note: The actual argument to delete is __builtin_coro_frame with the promise as parameter */
  operator delete(static_cast<void *>(__f));
}


int main()
{
  try 
  {
    Generator<unsigned long> gen = fibonacci_sequence(10);
    for(int j = 0; gen.operator bool(); ++j) {
      std::operator<<(std::operator<<(std::operator<<(std::cout, "fib(").operator<<(j), ")=").operator<<(gen.operator()()), '\n');
    }
    
  } catch(const std::exception & ex) {
    std::operator<<(std::operator<<(std::operator<<(std::cerr, "Exception: "), ex.what()), '\n');
  } catch(...) {
    std::operator<<(std::cerr, "Unknown exception.\n");
  }
  return 0;
}

```

## 参考链接
[Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)