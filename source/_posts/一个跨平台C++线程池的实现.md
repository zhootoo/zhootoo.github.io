---
title: 一个跨平台C++线程池的实现
date: 2026-10-31 12:32:20
categories: [线程池]
tags:
- C++
- 多线程
- 线程池
---

## 线程池的设计

在设计一个线程池之前，我们首先需要了解线程池大概有哪些部分组成：
1- 首先肯定是要有一定数量的worker线程，这些线程负责执行任务。他们需要从一个任务队列中取取出一个任务，然后执行这个任务。
2- 其次是任务队列，用于存放待执行的任务，worker也可以从这里面取任务。
3- 合理的同步机制，发布任务的时候和执行任务的时候需要一定的同步机制。

以上大概就是一个简单的线程池的组成部分了，具体的示意如下：

{% asset_img 1.png 线程池示意图 %}

具体实现如下，详细解释见注释

```C++
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <condition_variable>
#include <queue>
#include <string>
#include <chrono>
#include <functional>
class ThreadPool
{
    public:
    ThreadPool(int numThreads):stop(false){//构造函数指定worker数量
        for(int i=0;i<numThreads;i++)
        {
            threads.emplace_back([this](){
                while(true)//worker不能停
                {
                    std::unique_lock<std::mutex> lock(mtx);
                    condition.wait(lock,[this](){
                        return !tasks.empty() || stop;//等待任务，或者终止信号
                    });
                    
                    if(stop && tasks.empty())
                    {
                        return;//只有全部任务做完了才终止
                    }
                    auto task(std::move(tasks.front()));
                    tasks.pop();
                    mtx.unlock();
                    task();
                }
            });
        }
    }
    
    ~ThreadPool()
    {
        {
            std::unique_lock<std::mutex> lock(mtx);
            stop = true;
        }
        condition.notify_all();
        for(auto& t : threads)
        {
            t.join();//等待所有线程结束
        }
    }
    template<typename F, typename... Args>
    void enqueue(F&& f, Args&&... args)//万能引用，F&&总是被推导为正确的左值或者右值引用数据类型
    {
        std::function<void()> task = \
        std::bind(std::forward<F>(f),std::forward<Args>(args)...);//完美转发，
        //具体细节可以看完美转发和移动语义
        {
            std::unique_lock<std::mutex>lock(mtx);
            tasks.emplace(std::move(task));
        }
        condition.notify_one();
    }
    private:
    std::vector<std::thread> threads;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable condition;
    bool stop;//线程池何时终止
};

void lockPrint(const std::string& info)
{
    static std::mutex printMtx;//保证打印有序
    std::lock_guard<std::mutex> lk(printMtx);
    std::cout<<info<<std::endl;
}
int main()
{
    ThreadPool pool(4);
    for(int i=0;i<10;i++)
    {
        pool.enqueue([i]{
            lockPrint("task "+std::to_string(i)+" running");
            std::this_thread::sleep_for(std::chrono::seconds(2));
            lockPrint("task "+std::to_string(i)+" done");
        });
    }
}
```


