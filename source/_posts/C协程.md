---
title: C++协程
date: 2026-10-01 13:32:20
categories: [多线程]
tags:
- C++
- 多线程
- 协程
---

## 写在前面

C++20 引入了协程（coroutine）。它是这套标准里"语言机制最完整、标准库支持最少"的一个特性：
编译器只负责把函数改写成可以暂停/恢复的形态，**至于暂停之后谁来恢复、返回值怎么存、异常怎么传，
全部由使用者自己实现**。这也是为什么很多人看完语法依然不知道协程该怎么用。

本文从"协程是什么 → 解决什么问题 → 三个关键字 → Awaitable → promise_type → 实战"这条主线展开，
所有示例代码都可以直接编译运行（`g++ -std=c++20` / `clang++ -std=c++20` / MSVC `/std:c++20`）。

---

## 一、什么是协程

### 1.1 普通函数与协程的区别

普通函数的生命周期是"调用 → 执行 → 返回 → 栈帧销毁"，中间没有任何可以"定格"的时刻。
你无法在函数执行到一半时把它冻住、去干别的事、再从冻住的地方接着跑。

协程则是一个**可以暂停和恢复执行的函数**。

- 普通函数：状态完全依赖线程栈，返回即销毁，天然与"这一次调用"绑定。
- 协程：运行状态需要保存在一块由编译器生成的**协程帧（coroutine frame）**里，这块内存通常在**堆**上分配。
  恢复协程时，把帧里的状态重新装载起来继续执行；挂起时，把当前状态写回帧里。

所以协程是**和线程解耦**的：线程 A 把协程跑到一半挂起，线程 A 销毁了，线程 B 依然可以恢复这个协程并接着上次的
状态继续执行（前提是恢复时机的同步由调用者负责）。

### 1.2 有栈协程与无栈协程

| | 有栈协程（stackful） | 无栈协程（stackless，C++20） |
| --- | --- | --- |
| 代表实现 | `ucontext`、`boost.context`、libco、Go goroutine | C++20 coroutine、Rust async、JS async |
| 独立栈 | 每个协程有自己的栈（通常 KB 级） | 没有独立栈，只保存协程帧 |
| 能挂起的位置 | 任意函数调用深度 | 只能在协程函数体的 `co_await` 点 |
| 切换代价 | 保存/切换整个栈指针 | 保存少量寄存器 + 跳转 |
| 内存占用 | 大（协程数 × 栈大小） | 小（与局部变量数量相关） |
| 传染性 | 无，普通函数也能被挂起 | 有，`co_await` 会沿着调用链"传染" |

"无栈"并不是真的不用栈，而是**协程不拥有自己的栈**——它在哪个线程上被恢复，就用那个线程的栈。
代价是：你无法在 `co_await` 之外的地方（比如一个普通函数的中间）挂起；
想让"下层函数"也能挂起，下层函数自己也必须写成协程，并在上层 `co_await` 它。

### 1.3 协程帧里到底存了什么

编译器为每个协程生成一个结构体，大致包含：

- **promise 对象**：协程的"控制中枢"，见第五章；
- **形参**：按值拷贝进帧（引用形参拷贝的是引用本身）；
- **跨越挂起点的局部变量**：`int total` 这类在 `co_await` 前后都要用的变量；
- **awaiter 临时对象**：`co_await` 表达式产生的 awaiter 必须活到恢复之后，否则恢复时就是野指针；
- **挂起点索引**：恢复时根据它决定"从哪继续执行"；
- **`resume` / `destroy` 函数指针**：不同类型擦除后的 `coroutine_handle` 靠它调用真正的恢复逻辑。

因为帧大小在**编译期就完全确定**，所以只需要一次 `operator new(size)` 就能分配出来（也可以由你在
`promise_type` 里重载 `operator new` 走内存池）。

同时编译器还会生成一个"恢复函数"，形如：

```cpp
void Frame_resume(Frame* frame)
{
    switch (frame->suspend_point)   // 上次挂在哪里
    {
    case 0: goto resume_point_0;
    case 1: goto resume_point_1;
    // ...
    }
}
```

以及一个"销毁函数"，负责析构**已经构造过**的局部变量和 promise，然后释放帧内存。
这就是为什么帧里必须记录挂起点的索引——它同时决定了析构时要清理哪些对象。

### 1.4 协程与线程的关系

- 协程没有线程亲和性（thread affinity）：可以在 A 线程挂起、在 B 线程恢复。
- 但**同一时刻只能有一个线程在恢复它**。两个线程同时 `resume()` 同一个协程是数据竞争，属于未定义行为。
- 挂起期间协程帧是完全"静止"的，读写它不需要加锁；但"何时恢复"这件事必须由你的调度逻辑保证顺序，
  否则 `await_resume()` 读到的可能是还没写完的数据。

换句话说：**协程负责"状态保存"，线程/调度器负责"什么时候跑"**。这两件事是正交的。

### 1.5 三种并发模型的对比

| | 线程 | 回调/事件驱动 | C++20 协程 |
| --- | --- | --- | --- |
| 代码形态 | 顺序，阻塞 | 分散的回调 | 顺序，可挂起 |
| 切换开销 | 内核调度，微秒级 | 极低 | 用户态跳转，纳秒级 |
| 单机数量级 | 千级 | 万级以上 | 万级以上 |
| 栈/内存 | 默认 MB 级栈 | 无 | 协程帧，几十~几百字节 |
| 表达复杂流程 | 容易 | 很难 | 容易 |
| 数据竞争 | 需要大量同步原语 | 单线程模型下简单 | 单线程模型下简单 |

---

## 二、协程解决了什么问题

### 2.1 回调地狱

先看一个回调式的异步读：

```cpp
#include <chrono>
#include <functional>
#include <future>
#include <iostream>
#include <memory>
#include <thread>
#include <utility>

class IntReader
{
public:
    static void BeginRead(std::function<void(int)> callback)
    {
        std::thread([cb = std::move(callback)] {
            std::this_thread::sleep_for(std::chrono::seconds(1)); // 模拟耗时 IO
            cb(10);
        }).detach();
    }
};

void PrintInt()
{
    IntReader::BeginRead([](int value) {
        std::cout << "value: " << value << std::endl;
    });
}
```

> 注：`PrintInt` 返回后主线程若立刻退出，进程结束会把后台线程一起带走，这里省略了等待逻辑。

只读一次还好。如果现在要**异步读三个值，求它们的和**，代码就会变成这样：

```cpp
void PrintSum()
{
    // 回调版必须把状态放到堆上保活：回调到来时 PrintSum 早就返回了
    auto sum  = std::make_shared<int>(0);
    auto done = std::make_shared<std::promise<void>>();

    IntReader::BeginRead([sum, done](int a) {
        *sum += a;
        IntReader::BeginRead([sum, done](int b) {
            *sum += b;
            IntReader::BeginRead([sum, done](int c) {
                *sum += c;
                std::cout << "sum: " << *sum << std::endl;
                done->set_value();
            });
        });
    });

    done->get_future().wait(); // 还得自己想办法把"异步"兜回"等待"
}
```

这就是所谓的**回调地狱（callback hell）**。它的问题不只是"丑"：

1. **嵌套层数随步骤数线性增长**，三步已经三层，三十步不可维护；
2. **状态生命周期要手工管理**：`sum` 必须 `shared_ptr` 保活，稍不注意就是悬空引用；
3. **错误处理到处重复**：每个回调都要写一遍错误分支，且无法用一个 `try/catch` 覆盖整条链路；
4. **无法表达循环**：想"读 N 次"，只能靠递归或额外状态；
5. **取消、超时、并发控制**只能靠外部标志位层层传递。

### 2.2 手写状态机

另一条路是把流程拆成显式状态机：

```cpp
class SumStateMachine
{
public:
    void Start() { BeginReadA(); }

private:
    void BeginReadA()
    {
        IntReader::BeginRead([this](int v) { a_ = v; BeginReadB(); });
    }
    void BeginReadB()
    {
        IntReader::BeginRead([this](int v) { b_ = v; BeginReadC(); });
    }
    void BeginReadC()
    {
        IntReader::BeginRead([this](int v) {
            std::cout << "sum: " << a_ + b_ + v << std::endl;
        });
    }

    int a_ = 0;
    int b_ = 0;
};
```

状态机把"每一步的结果"变成了成员变量，但流程被切碎到多个函数里，
**读代码的人需要自己在大脑里把这些碎片拼回原本的顺序**。分支一多（比如"失败重试""超时就换源"），
状态数量会指数级膨胀。

### 2.3 同步的写法，异步的执行

用协程重写同一段逻辑：

```cpp
Task<int> PrintSum()
{
    int total = 0;
    total += co_await ReadInt();
    total += co_await ReadInt();
    total += co_await ReadInt();
    co_return total;
}
```

到这里应该能直观感受到区别：**代码是顺序写的，执行是非阻塞的**。
`co_await` 的意思是"挂起当前协程，等这个操作完成后再从这里继续"，
挂起期间线程并没有被阻塞，可以去跑别的协程。

这就是协程最核心的价值：**把异步流程的"控制流"重新变回线性**。
它也顺带解决了回调版的所有痛点——状态就是普通的局部变量（编译器帮你搬进帧里），
错误处理就是普通的 `try/catch`，循环就是普通的 `for`。

### 2.4 协程不解决什么

- **不是并行计算工具**：协程是"并发（concurrent）"不是"并行（parallel）"，一个线程上的协程还是串行执行。
  CPU 密集任务请用线程池。
- **不会自动变快**：它优化的是代码结构和内存占用，不是吞吐本身。
- **不能脱离调度器**：没人 `resume()`，协程永远停在原地。
- **不是线程的替代品**：多核并行仍然要靠线程。

---

## 三、C++20 协程的三个关键字

### 3.1 `co_await` / `co_yield` / `co_return`

只要函数体里出现了这三个关键字中的任意一个，这个函数**就自动变成协程**
（哪怕它只是 `co_return;` 一下就结束）：

| 关键字 | 含义 | 典型用途 |
| --- | --- | --- |
| `co_await expr` | 挂起，等 `expr` 表示的操作完成，取回结果 | 等待异步 IO、等待另一个协程 |
| `co_yield expr` | 产出一个值并挂起，下次恢复时继续 | 生成器、惰性序列 |
| `co_return [expr]` | 结束协程并返回值 | 协程的结果 |

三者本质上是同一套机制的三层包装：

```cpp
co_yield expr;   // 等价于  co_await promise.yield_value(expr);
co_return expr;  // 等价于  promise.return_value(expr); 然后走到 final_suspend
co_return;       // 等价于  promise.return_void();     然后走到 final_suspend
```

### 3.2 编译器把协程改写成了什么

以一个 `Task<int>` 协程为例，编译器大致会改写成下面的结构（伪代码，省略细节）：

```cpp
Task<int> SumThree()
{
    // ① 在堆（或栈，见 8.2）上分配协程帧：promise + 形参副本 + 局部变量 + awaiter
    auto* frame = new Frame{};
    auto& promise = frame->promise;

    // ② 先拿到返回给调用者的对象，但此时还不会返回
    Task<int> ret = promise.get_return_object();

    // ③ 初始挂起点：决定"协程是否立刻开跑"
    co_await promise.initial_suspend();

    try
    {
        int total = 0;

        // ④ 每个 co_await 变成一个"断点"，索引写进帧里
        {
            AsyncRead aw{};                 // awaiter 也活在被撑开的帧里
            frame->suspend_point = 1;
            if (!aw.await_ready())
                co_await aw.await_suspend(frame->handle);   // 在这里挂起
        // resume_point_1:  ← 恢复后从这里继续
            total += aw.await_resume();
        }
        // ……另外两次 co_await 同理……

        promise.return_value(total);        // ⑤ co_return 的落点
    }
    catch (...)
    {
        promise.unhandled_exception();      // ⑥ 统一兜住未捕获的异常
    }

    co_await promise.final_suspend();       // ⑦ 终结挂起点
    return ret;
}
```

关键点：**协程体被包在 `try` 块里**，所以协程里抛出的异常一定会走到 `unhandled_exception()`，
由你的 `promise_type` 决定是存起来、还是直接终止。

### 3.3 协程的一生

1. 分配协程帧（`operator new`）；
2. 把形参拷贝进帧；
3. 构造 promise 对象；
4. 调用 `promise.get_return_object()` 得到返回给调用者的对象；
5. `co_await promise.initial_suspend()`
   - 返回 `suspend_always` → 立刻挂起，控制权回到调用者（**lazy**）
   - 返回 `suspend_never` → 继续执行，直到第一个真正的挂起点（**eager**）
6. 执行函数体，遇到 `co_await` 就按第四章的规则挂起/恢复；
7. 遇到 `co_return` → `return_value()` / `return_void()`；遇到异常 → `unhandled_exception()`；
8. `co_await promise.final_suspend()`，停在终结挂起点；
9. 由持有 `coroutine_handle` 的人调用 `destroy()` 释放帧（若 `final_suspend` 返回 `suspend_never`，
   帧会自动销毁，见 5.3 的陷阱）。

---

## 四、`co_await` 与 Awaitable

### 4.1 Awaitable 三件套

`co_await` 的作用就是"等一等"：暂停协程，等条件满足再回来。
但编译器不可能知道"任意一个自定义类型什么时候算完成"，所以它要求被 `co_await` 的类型遵守一套约定，
这个约定叫 **Awaitable**。

编译器对 `co_await expr` 的求值顺序是：

1. 若 `promise_type` 有 `await_transform()`，先做 `expr = promise.await_transform(expr)`；
2. 若 `expr` 有 `operator co_await()`（成员或自由函数），则 `awaiter = expr.operator co_await()`；
3. 否则若 `expr` 本身就是一个 awaiter，则 `awaiter = expr`；
4. 否则编译错误。

而一个 **awaiter** 需要提供三个成员：

| 成员 | 返回值 | 作用 |
| --- | --- | --- |
| `await_ready()` | `bool` | `true` 表示"已经就绪，不用挂起"，直接跳到 `await_resume()` |
| `await_suspend(handle)` | `void` / `bool` / `coroutine_handle<Z>` | 挂起时被调用，负责注册回调、保存 handle、安排"谁来恢复我" |
| `await_resume()` | 任意类型 | 恢复后调用，其返回值就是整个 `co_await` 表达式的值 |

`std::suspend_always` 和 `std::suspend_never` 就是标准库提供的两个现成 awaiter：

```cpp
struct suspend_always
{
    bool await_ready() const noexcept { return false; }   // 永远挂起
    void await_suspend(std::coroutine_handle<>) const noexcept {}
    void await_resume() const noexcept {}
};

struct suspend_never
{
    bool await_ready() const noexcept { return true; }    // 从不挂起
    void await_suspend(std::coroutine_handle<>) const noexcept {}
    void await_resume() const noexcept {}
};
```

### 4.2 `await_suspend` 的三种返回值

这是协程里最容易被忽略、却最重要的细节：

| 返回类型 | 语义 |
| --- | --- |
| `void` | 挂起成功后，控制权回到"恢复者"（调用者或调度器），由它决定接下来跑什么 |
| `bool` | `true` 正常挂起；`false` **不挂起**，直接执行 `await_resume()`（常用于"检查后发现已经完成，就不用挂起"） |
| `coroutine_handle<Z>` | **对称转移（symmetric transfer）**：当前协程挂起，直接把控制权尾调用给 `Z`，不回到调用者的栈帧。这样"协程 A 恢复协程 B"不会让调用栈无限增长，深层协程链也不会爆栈 |

返回 `bool` 的写法在实现 `await_ready()` 无法预知的情况下很有用，例如"先尝试加锁，失败才挂起"：

```cpp
bool await_suspend(std::coroutine_handle<> h)
{
    if (mutex_.try_lock()) return false;   // 抢到了，不用挂起
    waiters_.push_back(h);                 // 抢不到，登记后挂起
    return true;
}
```

### 4.3 `co_await` 的完整执行流程

```text
co_await expr
   │
   ├─ 求 awaiter（await_transform / operator co_await / 自身）
   │
   ├─ await_ready() == true ──────────────────────► 直接执行 await_resume()，不挂起
   │
   └─ await_ready() == false
          │
          └─ r = await_suspend(handle)
                 ├─ void      → 协程挂起，控制权回到调用者/调度器
                 ├─ false     → 不挂起，继续执行 await_resume()
                 └─ handle Z  → 挂起当前协程，跳到 Z 继续执行（对称转移）
                                    │
                       某个线程调用 handle.resume()
                                    │
                              ──► await_resume() 求值，作为 co_await 的结果
```

### 4.4 手写一个异步读操作

下面把 2.1 里的 `IntReader` 改造成可被 `co_await` 的 awaiter：

```cpp
#include <chrono>
#include <coroutine>
#include <thread>

class AsyncRead
{
public:
    bool await_ready() const noexcept
    {
        return false;   // 数据还没读到，必须挂起
    }

    void await_suspend(std::coroutine_handle<> h)
    {
        // 真实项目里这里是 epoll/io_uring/线程池的任务投递；
        // 这里用一个 detach 线程模拟耗时 IO
        std::thread([this, h] {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            value_ = 10;
            h.resume();          // 数据就绪，恢复协程
        }).detach();
    }

    int await_resume() const noexcept
    {
        return value_;           // 成为 co_await 表达式的结果
    }

private:
    int value_ = 0;
};
```

三点说明：

- `await_suspend` 里捕获的 `this` 指向 awaiter 对象，而 awaiter 作为 `co_await` 的临时对象
  **活在协程帧里**，所以挂起期间它一直有效——这正是 1.3 里"awaiter 也要进帧"的原因。
- `h` 是当前协程的 handle（`coroutine_handle<>` 是类型擦除版），把它交给"完成时能通知我的人"，
  是异步转协程的**唯一接口**。
- 生产代码不要用 `detach` 线程：线程创建成本高，且无法控制并发度。正确做法是把 `h` 塞进
  IO 多路复用的完成队列。

---

## 五、`promise_type`：协程的控制中枢

### 5.1 `promise_type` 必须提供的接口

协程函数的**返回类型**里必须能找到 `promise_type` 这个成员类型
（或者你特化了 `std::coroutine_traits<...>`）。编译器通过它来驱动整个协程：

| 接口 | 是否必须 | 作用 |
| --- | --- | --- |
| `promise_type` | 必须 | 帧内 promise 对象的类型 |
| `get_return_object()` | 必须 | 构造返回给调用者的对象（通常持有 handle） |
| `initial_suspend()` | 必须 | 初始挂起点，返回 awaiter，决定 lazy/eager |
| `final_suspend()` | 必须 | 终结挂起点，返回 awaiter，决定帧何时销毁 |
| `return_void()` / `return_value(v)` | 二选一 | `co_return` 的行为，**不能同时存在** |
| `unhandled_exception()` | 必须 | 协程内未捕获异常的处理方式 |
| `yield_value(v)` | `co_yield` 时必须 | 产出一个值，返回 awaiter |
| `await_transform(e)` | 可选 | 改写 `co_await` 的操作数（例如强制协程上下文） |
| `operator new / delete` | 可选 | 自定义协程帧的分配策略 |

这也是 C++20 协程"只有语言机制、没有标准库"的体现：**上面这些全部要你自己写**。

### 5.2 给协程一个返回类型：`Task<T>`

`void` 是没有 `promise_type` 的，所以 `void` 函数**不能是协程**——这也是很多人第一个编译错误。
我们需要一个专门的返回类型，习惯上叫 `Task`（或 `task`、`lazy`）：

```cpp
#include <coroutine>
#include <exception>
#include <utility>

template <typename T>
class Task
{
public:
    struct promise_type
    {
        T value_{};                          // co_return 的值存这里
        std::exception_ptr error_{};         // 异常也存这里
        std::coroutine_handle<> continuation_{}; // 谁在等我（见 5.4）

        Task get_return_object()
        {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        // 惰性启动：创建 Task 时协程一行都不执行
        std::suspend_always initial_suspend() noexcept { return {}; }

        // 结束时挂住，让 Task 有机会读取结果并销毁帧
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T value) { value_ = std::move(value); }

        void unhandled_exception() noexcept { error_ = std::current_exception(); }
    };

    explicit Task(std::coroutine_handle<promise_type> handle) noexcept
        : handle_(handle) {}

    Task(Task&& other) noexcept
        : handle_(std::exchange(other.handle_, std::coroutine_handle<promise_type>{})) {}

    Task& operator=(Task&& other) noexcept
    {
        if (this != &other)
        {
            if (handle_) handle_.destroy();
            handle_ = std::exchange(other.handle_, std::coroutine_handle<promise_type>{});
        }
        return *this;
    }

    Task(const Task&) = delete;
    Task& operator=(const Task&) = delete;

    ~Task()
    {
        if (handle_) handle_.destroy();      // RAII：谁持有谁负责销毁帧
    }

    void resume() const { handle_.resume(); }
    bool done() const { return handle_.done(); }

    T result()
    {
        if (handle_.promise().error_)
            std::rethrow_exception(handle_.promise().error_);  // 异常穿透到等待者
        return std::move(handle_.promise().value_);
    }

private:
    std::coroutine_handle<promise_type> handle_{};
};
```

几个设计要点：

- **`Task` 必须是 move-only**：它持有一个协程帧的所有权，拷贝会导致重复 `destroy()`。
- **`initial_suspend` 返回 `suspend_always`** 是最常见的做法（惰性）：
  返回 `Task` 时协程还没跑，你可以在真正执行前把它塞进调度队列、组合、取消，
  也能避免"构造过程中协程立刻重入"带来的时序问题。
  如果想"创建即执行"，用 `suspend_never`。
- **`result()` 里检查 `error_`**：这样协程内的异常会沿着 `co_await` 链向上传播，
  写法与同步代码的 `try/catch` 完全一致。

### 5.3 `final_suspend` 与生命周期陷阱

`final_suspend` 的返回值决定了帧在协程结束后的命运：

| `final_suspend` 返回 | 结果 |
| --- | --- |
| `suspend_always` | 协程停在终结挂起点，`handle.done() == true`，**帧仍然有效**，必须手动 `destroy()` |
| `suspend_never` | 协程结束时帧**立即被销毁**，`handle` 立刻变成悬空指针 |

因此下面这样写是危险的：

```cpp
std::suspend_never final_suspend() noexcept { return {}; }  // 千万别这么写

// 调用方：
auto t = SomeCoroutine();   // 跑完就自动销毁了
t.result();                 // 悬空 handle → 未定义行为
```

几乎所有手写 `Task` 都用 `suspend_always`，把"销毁"交给 `Task` 的析构函数（RAII）。
代价是**必须保证 `Task` 一定被销毁**，否则就是内存泄漏——包括在 `co_await` 表达式里产生临时
`Task` 的场景。

相似的问题还有两类：

- **协程帧被销毁后仍有线程持有 handle**：`await_suspend` 里 `detach` 出去的线程在协程被
  `destroy()` 之后才 `resume()`，是典型的 use-after-free。
- **引用形参**：形参按值拷贝进帧，但如果形参是引用/指针，被引用对象的生命周期仍需由你保证。

### 5.4 协程套协程：`co_await` 一个 `Task`

真实代码里协程会互相调用。让 `Task` 可以被 `co_await`，需要两件事：

**① 给 `Task` 加 `operator co_await()`，返回一个 awaiter：**

```cpp
struct Awaiter
{
    std::coroutine_handle<promise_type> child_;

    bool await_ready() const noexcept { return false; }

    // 返回 handle → 对称转移：不回到父协程，直接开跑子协程
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> parent) noexcept
    {
        child_.promise().continuation_ = parent;   // 子协程结束后叫醒父协程
        return child_;
    }

    T await_resume()
    {
        auto& p = child_.promise();
        if (p.error_) std::rethrow_exception(p.error_);
        return std::move(p.value_);
    }
};

Awaiter operator co_await() const noexcept { return Awaiter{handle_}; }
```

**② 让 `final_suspend` 返回一个"唤醒 continuation"的 awaiter：**

```cpp
struct FinalAwaiter
{
    bool await_ready() const noexcept { return false; }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<promise_type> h) noexcept
    {
        if (auto next = h.promise().continuation_)
            return next;                  // 尾调用回父协程，栈不增长
        return std::noop_coroutine();     // 没人等，交给调度器
    }

    void await_resume() const noexcept {}
};

// promise_type 里把 suspend_always 换掉
FinalAwaiter final_suspend() noexcept { return {}; }
```

`suspend_always` 换成 `FinalAwaiter` 之后，协程结束时**不会**回到 `resume()` 的调用点，
而是直接跳到等待它的父协程——这就是对称转移的实际用法，也是实现零开销协程链的关键。

---

## 六、`co_return`：让协程返回值

### 6.1 `return_value` 与 `return_void`

- `co_return expr;` → `promise.return_value(expr)`，`expr` 会转换成 `return_value` 的形参类型；
- `co_return;` → `promise.return_void()`；
- 函数体走到末尾自然结束（没有任何 `co_return`）→ 等价于 `co_return;`，同样要求 `return_void()`；
- `return_void` 与 `return_value` **不能同时定义**；
- 协程里**不能出现普通的 `return` 语句**（`return;` 也不行）。

所以 `Task<T>` 用 `return_value`，而 `Task<void>` 只需要换个函数即可，其余结构完全一样：

```cpp
template <>
class Task<void>
{
public:
    struct promise_type
    {
        std::exception_ptr error_{};

        Task get_return_object()
        {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_void() noexcept {}                                   // 只是这里不同
        void unhandled_exception() noexcept { error_ = std::current_exception(); }
    };

    // 其余成员（handle 管理、resume/done 等）与 Task<T> 一致，result() 改为检查异常即可
};
```

### 6.2 异常处理

因为协程体被包在 `try` 块里，任何逃逸出函数体的异常都会进入 `unhandled_exception()`。
通常有两种策略：

```cpp
// 策略一：存起来，等 await_resume / result() 时重新抛出（推荐，异常可传播）
void unhandled_exception() noexcept { error_ = std::current_exception(); }

// 策略二：直接终止（适合"写错就该崩"的场景，零开销）
void unhandled_exception() noexcept { std::terminate(); }
```

注意：`unhandled_exception()` 本身**不是协程**，里面不能 `co_await`；
处理完异常后执行流会继续走到 `final_suspend()`。

### 6.3 完整示例：用协程重写异步求和

把 §4.4 的 `AsyncRead`、§5.2 的 `Task<T>` 和下面这段拼在一起，就是一个完整可编译的程序：

```cpp
#include <chrono>
#include <coroutine>
#include <exception>
#include <iostream>
#include <thread>
#include <utility>

// 1. Task<T>   —— 见 5.2
// 2. AsyncRead —— 见 4.4

Task<int> SumThree()
{
    int total = 0;

    total += co_await AsyncRead{};
    total += co_await AsyncRead{};
    total += co_await AsyncRead{};

    co_return total;
}

int main()
{
    Task<int> task = SumThree();   // 惰性：此时协程体一行都没执行
    task.resume();                 // 手动驱动；真实项目里交给事件循环

    // 示例用轮询等待结果，真实场景应由调度器驱动
    while (!task.done())
        std::this_thread::sleep_for(std::chrono::milliseconds(10));

    std::cout << "sum: " << task.result() << std::endl;
}
```

编译与运行：

```bash
g++ -std=c++20 -O2 -pthread coro.cpp -o coro && ./coro
# sum: 30
```

三个 `co_await` 各耗时 1 秒，**总耗时约 3 秒**——因为它们被串行等待。
如果你想让它们并行，需要把三个 `Task` 都启动起来再统一等待（这超出了本文范围，
但在 `cppcoro::when_all` 或 `std::execution` 的 sender/receiver 模型里就是标准能力）。

同一段逻辑用回调写，就是 §2.1 那段三层嵌套；用协程写，读起来和同步代码一模一样——
这就是"**同步的写法，异步的执行**"。

---

## 七、`co_yield`：生成器

### 7.1 `co_yield` 的本质

```cpp
co_yield expr;      // ≡ co_await promise.yield_value(expr);
```

`yield_value(expr)` 把值存进 promise，并返回一个 awaiter；只要这个 awaiter 是 `suspend_always`，
协程就会在这里挂起，等下一次 `resume()` 时从 `co_yield` 之后继续。

它把一个函数变成了"**可以多次返回、按需产出**"的序列生成器，
最典型的收益是：**生产逻辑可以写成顺序代码，而消费方按需取值**。

### 7.2 实现一个 `Generator`

```cpp
template <typename T>
class Generator
{
public:
    struct promise_type
    {
        T current_{};

        Generator get_return_object()
        {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }

        std::suspend_always yield_value(T value)
        {
            current_ = std::move(value);      // 把值交给外界
            return {};                        // 然后挂起
        }

        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { throw; } // 简单起见，直接向调用者抛
    };

    explicit Generator(std::coroutine_handle<promise_type> handle) noexcept
        : handle_(handle) {}
    Generator(Generator&& other) noexcept
        : handle_(std::exchange(other.handle_, std::coroutine_handle<promise_type>{})) {}
    Generator(const Generator&) = delete;
    Generator& operator=(const Generator&) = delete;
    ~Generator() { if (handle_) handle_.destroy(); }

    // 推进到下一个值；没有了返回 false
    bool next()
    {
        if (!handle_ || handle_.done()) return false;
        handle_.resume();
        return !handle_.done();
    }

    const T& value() const { return handle_.promise().current_; }

private:
    std::coroutine_handle<promise_type> handle_{};
};
```

使用时：

```cpp
Generator<int> Iota(int start, int step)
{
    int value = start;
    while (true)          // 无限序列也没问题，因为它是惰性的
    {
        co_yield value;
        value += step;
    }
}

int main()
{
    auto gen = Iota(0, 5);
    for (int i = 0; i < 5 && gen.next(); ++i)
        std::cout << gen.value() << ' ';
    std::cout << std::endl;
}
```

输出：

```text
0 5 10 15 20
```

### 7.3 为什么"惰性"很重要

- **内存 O(1)**：`Iota` 生成一亿个数也不需要一亿个元素的内存，用完即走；
- **可组合**：`take`、`filter`、`map` 这类操作可以一层层套在生成器上，每一次 `next()`
  才真正推进一格；
- **适合流式场景**：读日志、解析大文件、遍历语法树、分页拉取远端数据。

C++23 已经把这件事标准化了——`std::generator<T>` 可以直接写：

```cpp
#include <generator>

std::generator<int> Iota(int start, int step)
{
    int value = start;
    while (true)
    {
        co_yield value;
        value += step;
    }
}

// 还支持 range-for：for (int v : Iota(0, 5)) { ... }
```

---

## 八、性能、陷阱与工程实践

### 8.1 一次协程的开销

- **创建/销毁**：一次帧分配 + promise 构造 + 局部变量初始化；
- **挂起/恢复**：保存少量寄存器、写挂起点索引、跳转，**不进内核**。

量级上，协程切换通常在几十纳秒，而线程切换在微秒级，线程创建在数十微秒以上；
单机承载数万到数十万协程是常见场景。但这些都是实现相关的经验值，**以你的实测为准**。

### 8.2 堆分配与 HALO 优化

帧大小编译期已知，所以标准允许编译器做 **HALO（Heap Allocation eLision Optimization，
也叫 coroutine elision）**：当编译器能证明帧的生存期完全被调用者包住时，可以把帧放在调用者的栈上，
从而完全消除堆分配。

```text
co_await AsyncRead{};   // 帧随用随建随销，编译器有机会把它放在栈上
```

实践中 GCC/Clang/MSVC 在 `-O2` 下都能消除一部分，但对跨函数、跨调度器、被 `Task` 带出作用域的协程
通常无能为力。如果需要确定性，可以：

- 在 `promise_type` 里重载 `operator new/delete` 走内存池；
- 复用帧（`placement new` + 池化），但要注意 `destroy` 与复用之间的生命周期。

### 8.3 常见陷阱清单

1. **`void` 返回类型做协程**：编译错误，`void` 没有 `promise_type`。
2. **`final_suspend` 用 `suspend_never`**：帧被自动销毁，`handle` 变野指针。
3. **忘记 `destroy()`**：`suspend_always` 的帧不会被自动回收，泄漏。
4. **多线程同时 `resume()` 同一个协程**：数据竞争，UB。
5. **协程销毁后仍有线程持有 handle**：`detach` 线程、定时器、IO completion 都是常见来源。
6. **引用/指针形参**：帧里只存了引用本身，被引用对象的生命周期要自己保证。
7. **lambda 协程捕获 `this`**：对象先于协程销毁，恢复时就是 use-after-free。
8. **awaiter 里保存的裸引用**：awaiter 本身活在帧里没问题，但它引用的对象不一定活得久。
9. **忘记检查 `unhandled_exception`**：异常被静默吞掉，比崩溃更难查。

### 8.4 协程函数的限制

- 不能是 `constexpr` / `consteval` 函数；
- 不能是 `main`；
- 不能有可变参数（`...`）；
- 返回类型不能用 `auto` / `decltype(auto)` 推导（必须能静态确定 `promise_type`）；
- 构造函数、析构函数不能是协程；
- 不能包含普通 `return` 语句；
- lambda 可以是协程（C++20 起），此时 `promise_type` 由 lambda 的闭包类型（或 `coroutine_traits`）提供。

### 8.5 生态与选型

C++20 只给了机制，落地通常借助库：

| 库 | 特点 |
| --- | --- |
| `cppcoro` | 教学/原型友好：`task`、`generator`、`async_mutex`、`when_all` |
| Boost.Asio（`co_await` / `awaitable<T>`） | 网络 IO 的事实标准，`co_await socket.async_read_some(...)` |
| `folly::coro` | Facebook 生产级实现，配套 executor |
| `libunifex` / `std::execution`（P2300） | sender/receiver 模型，未来的标准异步框架 |
| `std::generator`（C++23） | 标准生成器，替代手写 `Generator` |

建议：**自己实现 `Task`/`Generator` 用来理解原理，真正写业务时用成熟库**——
手写实现里那些"忘记 destroy""忘了对称转移导致爆栈""异常传播没接上"的坑，库都已经踩过了。

---

## 九、小结

- 协程是**可以暂停/恢复的函数**，状态保存在编译器生成的**协程帧**里，与线程解耦；
- C++20 的协程是**无栈**的，只能在 `co_await` 点挂起，`co_await` 具有"传染性"；
- 它真正解决的是**异步流程的表达问题**：把回调地狱和手写状态机还原成线性代码，
  附带更好的局部性、更低的内存占用；
- 三个关键字背后是两套约定：**Awaitable（`await_ready`/`await_suspend`/`await_resume`）
  和 `promise_type`（`get_return_object`/`initial_suspend`/`final_suspend`/`return_*`/`unhandled_exception`）**；
- 协程**不是线程的替代品**，它需要调度器驱动；多核并行的部分仍然交给线程；
- 记住两条安全线：**`final_suspend` 用 `suspend_always` + RAII 管理帧**，**任何跨挂起的 async 回调都必须
  与帧的生命周期绑定**。

协程本身不复杂，复杂的是"谁拥有帧、谁负责恢复、异常怎么传"这三个问题的答案——
一旦这三点想清楚了，C++20 协程就只是"把函数拆成可以多次进出的片段"而已。
