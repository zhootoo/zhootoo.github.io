---
title: C++多线程In Practice
date: 2026-10-15 12:30:29
categories: [多线程]
tags: 
- C++
- 多线程
---

## 理解std::thread

我们可以看看C++ 11 std::thread的源码，windows平台：
```C++
class thread
{
    //....
    thread() noexcept() : _Thr{} {}
    explicit thread(_Fn&& _Fx, _Args&&... _Ax){
    //.....
    _Thr._Hnd = reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr,0,_Invoker_proc,_Decay_copied.get(),0,&_Thr._Id));
    }
    ~thread() noexcept{
        if(joinable())
        {
            _STD terminate();
        }
    }
};
```
从上面的代码来看，如果thread对象被析构的时候是joinable()的，意味着该 `std::thread` 对象拥有、关联着一个底层操作系统线程的执行实例，那么会直接terminate进程。

join():等待某个线程结束，调用方会被阻塞直到该线程结束。
detach():分离OS线程和thread对象的绑定关系，即使thread对象析构，也不会影响线程的执行。但是如果线程在主线程结束后仍然执行，进程会被终止，子线程会被粗暴kill，子线程的局部变量不会RAII清理。如果子线程访问了全部变量或者静态变量，那么主线程析构的时候可能会发生UB，或者崩溃。所以如果要使用detach()，就不要使用外部资源，而且要在主线程推出前合理退出。

##线程创建的方式
thread t(ThreadMain,101,var1,var2, Object1,...)//可以传递任意数量的参数,这里注意的是默认传入的参数是会被拷贝到新创建的线程中的。

如果传递指针或者引用，要确保所访问的对象在线程执行过程中是存在的没有被销毁。如果要传递的是引用，需要在参数加上std::ref()。

传入对象成员函数作为线程入口的方式：
```C++
class ThreadObj
{
public:
    void ThreadMain(int arg1, int arg2)
    {
        //....
    }
};

ThreadObj obj;
std::thread t(&ThreadObj::ThreadMain, &obj, 101, 202);

```

## 线程传参
需要说明传参的不同方式并给出具体的示例和输出

## 线程同步
需要说明同步的不同方式并给出具体的示例和输出

## 原子操作
需要说明原子操作的不同方式并给出具体的示例和输出

## promise和future
需要说明引入promise和future的原因，适用场景。并给出一个具体的示例。给出结果。

