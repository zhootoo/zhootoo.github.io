---
title: 聊聊 C++ 的 atomic 原子类型
date: 2026-11-2 11:30:29
categories: [多线程]
tags: 
- C++
- 多线程
- 原子类型
---

多线程碰到共享数据，大家条件反射就是 `std::mutex` 加锁。这当然没错，但有时候会觉得杀鸡用牛刀——一个全局计数器，两个线程各累加一百万次，就这么点事，每次自增都 lock/unlock 一圈，既浪费性能又把代码搞得很啰嗦。

C++11 引入的 `std::atomic` 就是为这种"简单变量"准备的。它比锁轻量得多，用起来也干净。这篇文章把 atomic 的用法、底层机制和几个容易踩的坑理一理。

## 先跑个例子感受下

两个线程，各对同一个计数器递增一百万次：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>
#include <chrono>

// 换成 mutex 版本就把 #if 改成 1
#if 0
std::mutex my_mutex;
int counter = 0;
void worker()
{
    for (int i = 0; i < 1000000; ++i)
    {
        my_mutex.lock();
        ++counter;
        my_mutex.unlock();
    }
}
#else
std::atomic<int> counter(0);
void worker()
{
    for (int i = 0; i < 1000000; ++i)
    {
        ++counter;
    }
}
#endif

int main()
{
    auto start = std::chrono::steady_clock::now();

    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();

    auto end = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "counter = " << counter << ", time = " << ms.count() << "ms" << std::endl;
    return 0;
}
```

跑一遍对比，atomic 版本又快又干净。道理也不复杂：mutex 的 lock/unlock 本身就有开销（系统调用、上下文切换的可能性），而 atomic 在多数情况下直接编译成一条 CPU 指令就搞定了。

**结论就两条：对简单场景，atomic 比 mutex 更高效、更简洁。**

## 什么是 atomic

`std::atomic<T>` 是一个类模板，它保证对 `T` 类型的单个变量，读和写都是原子操作——不会出现"写了一半被另一个线程读走"的情况。说白了，就是把一个普通变量包了一层，让所有对它的访问都变成不可分割的。

什么类型可以放进 `std::atomic`？基本上，**trivially copyable** 的类型都行——也就是那些能用 `memcpy` 正确拷贝的类型：

```cpp
std::atomic<char> ac('H');
std::atomic<int> ai(0);
std::atomic<bool> ab(false);
std::atomic<double> ad(3.14);
std::atomic<int*> ap(nullptr);   // 指针也行
```

自定义类型也支持，但有限制。如果类里写了自定义的拷贝构造/拷贝赋值/移动构造/移动赋值/析构函数，或者有虚函数、引用成员，那就没法放进 `std::atomic` 了，编译直接报错。

```cpp
struct Demo
{
    Demo(const Demo&) {}   // 自定义拷贝构造 → 编译失败
    int a;
    int b;
};

std::atomic<Demo> ao;  // error
```

记住一个简单的判据：能被 `memcpy` 正确拷贝的，才能放进 `atomic`。

## 基本操作

atomic 的操作接口不算多，把常用的几个过一遍：

```cpp
std::atomic<int> ai(0);

// 读
int val = ai.load();        // 原子地读
// 写
ai.store(20);               // 原子地写
ai = 20;                    // 赋值运算符也行

// 读旧值、写新值，返回旧值
int old_val = ai.exchange(30);

// 复合运算（整型支持）
ai += 10;    // 等价于 ai.fetch_add(10)
ai -= 10;    // 等价于 ai.fetch_sub(10)
ai |= 10;    // 等价于 ai.fetch_or(10)
ai &= 10;    // 等价于 ai.fetch_and(10)
ai ^= 10;    // 等价于 ai.fetch_xor(10)
++ai;
ai++;
```

有几点值得留意：

- `ai.load()` 本身是原子的，但把返回值赋给普通变量这个动作不是原子的。`int val = ai.load()` 里，load 是原子的，赋值给 `val` 不是。同理 `exchange` 的返回值赋值也不是。
- `std::atomic` 没有重载 `<<`，`cout << ai` 能编译是因为隐式转换成了 `int`。
- 不同类型支持的操作不一样。`int` 这类整型可以做加减和位运算；`double` 就没有 `fetch_add` 这些；指针类型支持 `++/--/+=/-=` 和 `fetch_add/fetch_sub`（做偏移）。

## CAS：compare_exchange

CAS（Compare-And-Swap）是原子操作里最核心的一个，也是实现无锁数据结构的基石。逻辑很简单：

> 原子地检查变量当前值是否等于预期值 `expected`，是就更新为 `desired` 并返回 `true`；否则把当前值写回 `expected` 并返回 `false`。

接口长这样：

```cpp
bool compare_exchange_strong(T& expected, T desired);
bool compare_exchange_weak(T& expected, T desired);
```

strong 和 weak 的区别：

- **strong**：只要值匹配就一定返回 `true`，不会有"虚假失败"。
- **weak**：即使值匹配了，也可能因为硬件层面的原因返回 `false`（spurious failure）。但在支持的平台上，weak 的实现往往更简单高效。

一般的经验法则是：**循环重试的场景用 weak，只试一次的场景用 strong。** 不过在不少平台上两者实现完全一样，性能也没差别。

看个实际例子——两个线程各循环 100 次，计数器到 20 就归零，否则加 1：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>
#include <mutex>

std::atomic<int> counter(0);
std::mutex my_mutex;   // 仅用来保护 cout 输出

void worker()
{
    for (int i = 0; i < 100; ++i)
    {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));

        int old_value = counter.load();
        int new_value;
        do {
            new_value = (old_value == 20) ? 0 : old_value + 1;
        } while (!counter.compare_exchange_strong(old_value, new_value));

        std::lock_guard<std::mutex> lock(my_mutex);
        std::cout << old_value << "\t" << std::this_thread::get_id() << std::endl;
    }
}

int main()
{
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
    return 0;
}
```

这个 `do { ... } while (!compare_exchange_strong(...))` 是 CAS 的经典模式。注意 `compare_exchange_strong` 失败时会自动把 `old_value` 更新为当前值，所以直接进下一轮重试就行，不用手动重新 load。

顺便说一句，这里的 mutex 只是为了让 `cout` 的输出不乱掉，跟计数器的同步无关。

## 底层是怎么实现的

atomic 的原子性有两种来源：

1. **CPU 硬件指令**。现代 CPU（x86、ARM 等）提供原子指令，对满足大小和对齐要求的类型可以直接做无锁的原子操作。
2. **内部互斥锁兜底**。当类型太大、不满足对齐要求、或者平台不支持对应的硬件指令时，atomic 内部会退化成一把锁来模拟原子性。正确性没问题，但性能就打了折扣。

怎么知道走的哪条路？调 `is_lock_free()`：

```cpp
std::atomic<int> ai(0);
std::cout << std::boolalpha << ai.is_lock_free() << std::endl;  // 通常 true
```

返回 `true` 就是硬件原子指令，`false` 就可能退化成了锁。

以 MSVC 为例，判断规则大致是：类型大小不超过 8 字节、且大小是 2 的幂（1、2、4、8）。来个反例：

```cpp
#pragma pack(push, 1)
struct Demo
{
    int a;    // 4 字节
    char c;   // 1 字节
};
#pragma pack(pop)

std::atomic<Demo> demo;
std::cout << std::boolalpha << demo.is_lock_free() << std::endl;  // false
```

`Demo` 被 pack 成 5 字节，小于 8 但不是 2 的幂，所以不是 lock-free 的。`is_lock_free()` 的结果跟平台、CPU 架构、编译器实现甚至实例的内存对齐都有关，不能想当然。

再说一次：`is_lock_free()` 返回 `false` 不代表会出错，只是内部可能走了锁的路径，性能差一些。

## 内存顺序：memory_order

到目前为止，atomic 解决的问题是"单次读写的原子性"。但在多线程环境下还有两个更隐蔽的坑：**指令重排**和**修改顺序不一致**。这就需要内存顺序来管了。

### 指令重排

编译器和 CPU 为了优化性能，可能调整指令的执行顺序，只要单线程的逻辑结果不变就行。单线程无所谓，多线程就可能出事：

```cpp
std::atomic<bool> ready{false};
int data = 0;

// 线程 A（写）
void writer()
{
    data = 42;
    ready.store(true);
}

// 线程 B（读）
void reader()
{
    while (!ready.load());
    assert(data == 42);   // 可能失败！
}
```

如果编译器把 `data = 42` 重排到 `ready.store(true)` 之后，reader 就可能看到 `ready == true` 但 `data` 还是 0。

C++ 提供了几种内存顺序来约束重排行为：

| 内存顺序 | 约束 |
|---|---|
| `memory_order_relaxed` | 不做任何约束，随便重排 |
| `memory_order_release` | 保证这条原子写之前的操作不被重排到它后面 |
| `memory_order_acquire` | 保证这条原子读之后的操作不被重排到它前面 |
| `memory_order_acq_rel` | 同时具备 release 和 acquire 的约束 |
| `memory_order_seq_cst` | 最强约束，且所有线程对修改的观察顺序一致（默认） |

用文字描述一下直观感受：

```cpp
A1; A2;
atomic_var.store(x, std::memory_order_release);  // A1、A2 不会跑到 store 后面
B1; B2;                                          // B1、B2 不受约束
```

```cpp
A1; A2;                                          // A1、A2 不受约束
atomic_var.load(std::memory_order_acquire);      // B1、B2 不会跑到 load 前面
B1; B2;
```

对上面的 writer/reader 问题，经典的修法就是写端用 `release`、读端用 `acquire`，形成 release-acquire 同步对：

```cpp
void writer()
{
    data = 42;
    ready.store(true, std::memory_order_release);
}

void reader()
{
    while (!ready.load(std::memory_order_acquire));
    assert(data == 42);   // 安全了
}
```

### 修改顺序一致性

还有一个更微妙的问题：原子操作保证每次读写的原子性，但不保证所有线程"看到"的修改顺序一致。

```cpp
std::atomic<int> x{0};

// 线程 T1
x.store(1, std::memory_order_release);
// 线程 T2
x.store(2, std::memory_order_release);

// 线程 A：可能读到 1 然后 2
// 线程 B：可能读到 2 然后 1
```

两个写线程分别写入 1 和 2，两个读线程可能对"谁先写"有不同的观察结果。如果你的程序依赖"全局统一的修改顺序"，这就出问题了。

解决方案是用 `memory_order_seq_cst`（Sequentially Consistent）。它在 release/acquire 的基础上再加一条约束：**所有线程对这些原子修改的观察顺序必须一致**。要么都看到 1→2，要么都看到 2→1，不会有分歧。

好消息是 `seq_cst` 就是默认值——不指定内存序的时候用的就是它。所以如果你不确定该用哪个，就别指定，用默认的。

## 几个实践建议

最后把散落各处的经验汇总一下：

**选型**：简单变量（计数器、标志位、指针）的并发读写，用 atomic 就对了。涉及多个变量的一致性、或者复杂的数据结构操作，老老实实上锁。别硬拿 atomic 去凑。

**类型限制**：放进 `atomic` 的类型必须是 trivially copyable 的。写了自定义拷贝/析构/虚函数的类就别想了。

**CAS 选型**：循环重试用 `compare_exchange_weak`，只试一次用 `compare_exchange_strong`。不确定就用 strong，至少语义上不会有虚假失败。

**性能评估**：关心性能的话，查一下 `is_lock_free()`。返回 `false` 说明这个类型的 atomic 可能退化成了锁，不是硬件级的。

**内存序**：默认的 `seq_cst` 最安全。对性能有极致要求时才考虑放宽到 `acquire`/`release`/`relaxed`，而且一定要想清楚重排的影响。没想清楚就别动。

**一个容易忽略的细节**：原子操作本身原子 ≠ 赋值原子。`int val = ai.load();` 这行里 load 是原子的，赋给 `val` 不是。不过只要 `val` 是线程局部的，这也无所谓。
