---
title: C++多线程In Practice
date: 2026-10-15 12:30:29
categories: [多线程]
tags: 
- C++
- 多线程
---

## 理解std::thread

很多人第一次写多线程代码，崩的地方不是业务逻辑，而是 `std::thread` 的析构。所以这章先不急着讲怎么用，而是看看它的底层机制，把"为什么会崩"搞明白。

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

写个最小例子复现一下：

```C++
#include <thread>
#include <iostream>

void Worker() {
    std::cout << "worker is running\n";
}

int main() {
    std::thread t(Worker);
    // 忘了 join() 或 detach()
    return 0;
}   // t 在这里析构，joinable() 为 true，std::terminate()，进程直接 abort
```

为什么标准库要设计得这么狠？想想另外两种选择就明白了：如果析构时自动 `join()`，那么一个本该很快返回的函数可能会在无法预期的位置卡住；如果自动 `detach()`，等于悄悄放出一个没人管的后台线程，出了问题更难查。标准库选择把决定权交给写代码的人——要么 `join()` 要么 `detach()`，不表态就崩给你看。这个"崩"其实是好事，比悄悄埋雷强多了。

**join()**：等待某个线程结束，调用方会被阻塞直到该线程结束。join 之后 `joinable()` 变为 false，thread 对象和底层 OS 线程的关联被解除，可以安全析构，也可以去接收/移动另一个线程。

**detach()**：分离OS线程和thread对象的绑定关系，即使thread对象析构，也不会影响线程的执行。但用之前得想清楚几件事：

- main 返回后进程退出，detach 出去的线程会被粗暴 kill，不会执行析构函数，线程内的局部变量不会走 RAII 清理。
- 如果线程访问了已经析构的对象（局部变量、生命周期先结束的全局对象），那就是 UB，可能崩溃，可能算出脏数据，也可能看起来一切正常——这恰恰是最危险的。

detach 的经典事故长这样：

```C++
#include <thread>
#include <chrono>
#include <iostream>

void Oops() {
    int local = 42;
    std::thread t([&local] {              // 按引用捕获了局部变量
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << local << std::endl;  // 一秒后 local 早就没了
    });
    t.detach();
}

int main() {
    Oops();   // Oops 返回，local 的生命周期结束
    std::this_thread::sleep_for(std::chrono::seconds(2));  // 等 detach 的线程作妖
}
```

所以如果要用 detach()，让线程只用自己拷贝进来的数据（值传递、std::move），或者通过 `std::shared_ptr` 持有共享数据，别碰外面的裸引用和裸指针，而且要保证线程在程序退出前合理收尾。能不 detach 就不 detach，这是实践中最省心的原则。

还有一个场景容易被忽略：join 之前函数中途抛了异常，join 就执行不到，析构照样 terminate。可以用 RAII 把线程包一层：

```C++
class ThreadGuard {
public:
    explicit ThreadGuard(std::thread t) : t_(std::move(t)) {}
    ~ThreadGuard() {
        if (t_.joinable()) {
            t_.join();
        }
    }
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;

private:
    std::thread t_;
};

void F() {
    std::thread t(Worker);
    ThreadGuard g(std::move(t));
    MightThrow();   // 即使这里抛异常，g 析构时也会 join
}
```

析构时检查 `joinable()` 是必要的——thread 被 join 或 move 之后就是"空"的，再 join 会抛 `system_error`。

## 线程创建的方式

创建线程的入口可以是普通函数、lambda、函数对象，也可以是成员函数，先汇总看一遍：

```C++
// 1. 普通函数
void Hello() { std::cout << "hello from function\n"; }
std::thread t1(Hello);

// 2. lambda，实际项目里最常用的写法
std::thread t2([] { std::cout << "hello from lambda\n"; });

// 3. 函数对象
struct Greeter {
    void operator()() const { std::cout << "hello from functor\n"; }
};
std::thread t3(Greeter{});
```

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

第一个参数是成员函数指针，第二个参数是"这个成员函数要在哪个对象上调用"，后面才是真正的参数。`&obj` 传的是指针，线程内部通过它去调 `obj.ThreadMain(101, 202)`，所以 obj 的生命周期必须覆盖线程的整个执行期。如果想更保险，也可以直接传对象本身：`std::thread t(&ThreadObj::ThreadMain, ThreadObj{}, 101, 202);`，这样线程内部会持有一份拷贝，代价是多一次拷贝构造。

最常用的还是带参形式：

```C++
thread t(ThreadMain, 101, var1, var2, Object1, ...)   // 可以传递任意数量的参数
```

这里注意的是默认传入的参数是会被拷贝到新创建的线程中的。验证一下：

```C++
int n = 100;
std::thread t([](int x) {
    std::cout << "thread sees x = " << x << '\n';
}, n);
n = 200;   // 改的是 main 这边的 n
t.join();
// 输出：thread sees x = 100
```

线程构造的那一刻参数就被拷走了，之后外面怎么改都不影响。如果传递指针或者引用，要确保所访问的对象在线程执行过程中是存在的没有被销毁。如果要传递的是引用，需要在参数加上std::ref()——细节下一章展开。

## 线程传参

传参看着简单，坑却是最多的，一个个过。

**值传递：最安全的方式**

```C++
void ByValue(std::string s) {
    std::cout << "thread own copy: " << s << '\n';
}

int main() {
    std::string msg = "hello";
    std::thread t(ByValue, msg);
    msg.clear();   // 不影响线程内部那份拷贝
    t.join();
    // 输出：thread own copy: hello
}
```

线程内部持有自己的一份拷贝，跟外部解耦，对象生命周期问题直接消失。缺点就是拷贝开销，传大对象时得掂量一下。

**引用传递：必须显式 std::ref()**

```C++
void ByRef(int& n) {
    n *= 2;
}

int main() {
    int n = 21;
    // std::thread t(ByRef, n);           // 编译不过！
    std::thread t(ByRef, std::ref(n));    // 正确
    t.join();
    std::cout << n << '\n';
    // 输出：42
}
```

直接传 `n` 为什么编译不过？因为 std::thread 构造时会把参数先拷贝到内部存储（所谓 decay-copy），新线程里执行函数时用的其实是这份内部拷贝，而一个临时拷贝没法绑定到非 const 引用上，所以直接报错。报错信息是一大坨模板展开，第一次见基本读不懂，记住这个现象就行：**非 const 引用参数不加 std::ref 就编不过，这不是 bug，是故意的保护**。

const 引用参数倒是能直接传（`void F(const std::string& s)`），但要注意线程里绑定的依然是内部拷贝，不是你原来那个对象。想省一次拷贝的话，用 `std::cref()` 传 const 引用，前提还是那句话——原对象活得比线程久。

**指针传递：能编译，但风险自负**

```C++
void ByPtr(int* p) { *p *= 2; }

int n = 21;
std::thread t(ByPtr, &n);   // 传的是地址
t.join();                   // join 之前 n 一直活着，没问题
```

指针本身也被拷贝了（拷贝的是地址值），但指向的对象线程够不够得着、活不活着，全靠你自己保证。

**char* 的隐蔽坑**

这个坑值得单独说。假设线程函数的参数是 `std::string`：

```C++
void F(const std::string& s);

void Oops(const char* text) {
    std::thread t(F, text);              // 危险！
    t.detach();
}
```

看起来传个 `const char*` 给 `const std::string&` 参数很自然，但 std::thread 拷贝的是指针本身，`std::string` 的构造发生在新线程里。如果这中间 text 指向的内容被修改或释放（比如 detach 之后缓冲区被回收），就是 UB。修法很简单，在当前线程就把转换做完：

```C++
std::thread t(F, std::string(text));   // 先构造好 string，再拷贝这个 string
```

**移动传递：独占资源**

`std::unique_ptr` 这类只能移动的类型，传给线程用 `std::move`：

```C++
#include <memory>

void Sink(std::unique_ptr<int> p) {
    std::cout << "got " << *p << '\n';
}

int main() {
    auto p = std::make_unique<int>(42);
    std::thread t(Sink, std::move(p));   // 所有权移交线程
    t.join();
    // 输出：got 42
}
```

move 之后线程内部独占这份资源，生命周期问题同样自动解决，还没有值传递的拷贝开销，传大对象时优先考虑这个。

总结成一句话：**能用值传递或 move 就优先用，要共享可变状态就 std::ref() 加生命周期保证**。

## 线程同步

多线程的大头来了。先看不同步会发生什么。

**数据竞争长什么样**

```C++
#include <thread>
#include <iostream>

int main() {
    int counter = 0;
    auto work = [&counter] {
        for (int i = 0; i < 100000; ++i) {
            ++counter;
        }
    };
    std::thread t1(work), t2(work);
    t1.join(); t2.join();
    std::cout << counter << '\n';
}
```

两个线程各加 10 万次，期望输出 200000，实际每次都不一样：

```
147382
189213
200000    // 偶尔运气好
155516
```

原因：`++counter` 在机器层面是"读内存、加一、写回"三步，两个线程交错执行时，A 读到 100，B 也读到 100，各自加一写回 101，两次自增丢了一次。严格来说这已经是数据竞争，属于未定义行为，能观察到"只是数字不对"已经算运气好了。

**mutex：最基本的修法**

```C++
#include <mutex>

int counter = 0;
std::mutex m;

auto work = [&] {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(m);   // 构造加锁，析构解锁
        ++counter;
    }
};
// 稳定输出：200000
```

`std::lock_guard` 是 RAII 风格，拿到锁的 scope 结束自动释放，中途抛异常也不会忘记 unlock。**不要手动调 lock()/unlock()**，漏一个路径就死锁。需要更灵活的控制（中途解锁、延迟加锁、配合条件变量）时用 `std::unique_lock`，日常场景 lock_guard 够了。

顺便一提，这个例子里 10 万次循环每次都锁一次开销不小，把锁挪到循环外面（一次锁做完所有自增）会快得多——锁的粒度是个永恒的权衡：粒度太细锁本身的开销上来，太粗并发度又下去。

**死锁**

两个线程、两把锁、顺序相反，教科书配置：

```C++
std::mutex m1, m2;

std::thread a([&] {
    std::lock_guard<std::mutex> l1(m1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));  // 放大竞态窗口
    std::lock_guard<std::mutex> l2(m2);   // 等 m2……
    std::cout << "a done\n";
});

std::thread b([&] {
    std::lock_guard<std::mutex> l2(m2);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> l1(m1);   // 等 m1……
    std::cout << "b done\n";
});
// 两个线程互相等对方手里的锁，谁也走不动，程序卡死，不输出任何东西
```

两个方向的传统解法：

1. 全局约定加锁顺序，所有线程都先 m1 后 m2，顺序一致就不会成环。
2. C++17 的 `std::scoped_lock`，一把锁多个互斥量，内部用死锁避免算法：

```C++
std::scoped_lock lock(m1, m2);   // 一次锁两个，不用操心顺序
```

**condition_variable：线程间等事件**

互斥量管"别一起动"，条件变量管"等你好了通知我"。经典的生产者-消费者骨架：

```C++
#include <condition_variable>
#include <queue>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;

// 消费者：等队列里有东西
std::thread consumer([&] {
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, [] { return !q.empty(); });   // 带谓词的 wait
    int item = q.front();
    q.pop();
    std::cout << "consumed " << item << '\n';
});

// 生产者：放数据，发通知
{
    std::lock_guard<std::mutex> lock(m);
    q.push(42);
}
cv.notify_one();

consumer.join();
// 输出：consumed 42
```

几个要点：

- `wait(lock, pred)` 里的谓词不可省。条件变量存在**虚假唤醒**（没人 notify 也可能醒），而且从被唤醒到重新拿到锁之间，条件可能又被别的线程改掉了。谓词版等价于"加锁 → 循环检查条件 → 不满足就继续睡"，手写就是 `while (!pred()) cv.wait(lock);`。
- 修改共享状态（push）之前要先拿锁，改完再 notify，顺序反了就有竞态。
- `notify_one` 叫醒一个，`notify_all` 全叫醒，按需选。

**读写锁：读多写少的场景**

配置数据这类"一群线程读、偶尔一个线程写"的场景，`std::shared_mutex`（C++17）比普通 mutex 并发度高：

```C++
#include <shared_mutex>

std::shared_mutex rw;
int config = 0;

// 读线程：共享锁，多个读者可以同时持有
{
    std::shared_lock<std::shared_mutex> lock(rw);
    std::cout << "config = " << config << '\n';
}

// 写线程：独占锁，写的时候谁都不许进
{
    std::unique_lock<std::shared_mutex> lock(rw);
    config = 42;
}
```

读写锁也不是免费午餐，读者太多时锁本身的管理开销可能超过普通 mutex，上线前拿真实负载压一压再定。

**一次性初始化**

顺带一提 `std::call_once`，多个线程同时跑到初始化逻辑时，保证只执行一次：

```C++
std::once_flag flag;

void InitOnce() {
    std::call_once(flag, [] { std::cout << "init\n"; });
}
```

不过单例这类场景，C++11 之后局部 static 变量本身就有线程安全保证（Magic Statics），`static Singleton instance;` 一行就够了，call_once 更多用在初始化时机不完全受控的地方。

## 原子操作

互斥量是通用方案，但锁不便宜：拿锁放锁本身有开销，还可能等待、可能死锁。如果临界区就是"改一个数"这种粒度，CPU 有专门的指令支持，这就是原子操作——**用一条不可分割的指令完成读改写，中间不可能被别的线程插进来**。

```C++
#include <atomic>

int main() {
    std::atomic<int> counter{0};
    auto work = [&counter] {
        for (int i = 0; i < 100000; ++i) {
            ++counter;   // 原子自增，不会丢更新
        }
    };
    std::thread t1(work), t2(work);
    t1.join(); t2.join();
    std::cout << counter << '\n';
    // 输出：200000（每次都是）
}
```

同样是两个线程各加 10 万次，换成 `std::atomic<int>` 结果就稳了。可以直接赋值和读取，用起来跟普通 int 差不多，限制是它没有普通的 `operator++` 之外的复合操作语义细节——比如 `fetch_add` 返回旧值：

```C++
std::atomic<int> n{0};
int old = n.fetch_add(5);   // 加 5，返回加之前的值
// old == 0, n == 5
```

想确认某个类型是不是真的无锁（而不是内部偷偷用了锁），问它：

```C++
std::atomic<int> a;
std::cout << a.is_lock_free() << '\n';   // x64 上输出 1（true）
```

机器字长以内的标量类型基本都是无锁的，超大的结构体就不一定了，内部可能还是挂了把锁。

有一句话必须说在前面：**原子操作保证的是单个操作的原子性，不是一组操作的一致性**。两个变量要一起保持一致，原子操作帮不了你，还是得靠锁，或者想办法把多个状态打包进一个原子变量。

**CAS：无锁算法的基石**

原子操作里最有意思的是 compare-and-exchange。思路是："我觉得这变量现在是 expected，是的话就换成 desired，不是的话把当前实际值告诉我，我重新想办法"。

```C++
std::atomic<int> value{0};
int expected = 0;   // 期望当前是 0

if (value.compare_exchange_strong(expected, 10)) {
    std::cout << "swapped, now value = " << value << '\n';
}
// 输出：swapped, now value = 10
```

失败时 expected 会被更新为当前实际值，所以重试循环里不用重新 load。用 CAS 可以写出一个无锁栈的 Push：

```C++
template <typename T>
class LockFreeStack {
    struct Node {
        T value;
        Node* next;
    };
    std::atomic<Node*> head_{nullptr};

public:
    void Push(const T& value) {
        Node* node = new Node{value, nullptr};
        node->next = head_.load();
        // CAS：head 还是我看到的那个，就换成新 node
        // 失败说明有别的线程抢先改了 head，node->next 已被更新为最新值，重试
        while (!head_.compare_exchange_weak(node->next, node)) {
        }
    }
};
```

`compare_exchange_weak` 相比 strong 可能"伪失败"——值明明相等也返回 false，所以它必须放在循环里用，换来的是某些平台上的更高性能；不循环的场景用 strong。另外无锁栈的 Pop 要面对内存回收的难题（什么时候 delete 节点才安全），比 Push 复杂一大截，这里先不展开，知道有这回事就行。

**内存序，先不慌**

原子操作还有个 memory_order 参数，一查资料满天飞的 acquire/release/relaxed 很容易劝退人。实践建议：**默认的 seq_cst（顺序一致性）先无脑用**，它是最强保证，所有原子操作存在全局统一的顺序，不会错。等 profiler 真的告诉你原子操作的内存屏障成了瓶颈，再去研究 relaxed 那一套也不迟。多数程序到不了那个阶段。

## promise和future

回到 `std::thread` 本身，它有两个先天缺陷：

1. **拿不到返回值**。线程函数 `return` 的结果直接被扔掉，调用方干瞪眼。
2. **异常传不回来**。线程里抛出未捕获的异常，直接 `std::terminate()`，进程说没就没，主线程连 try-catch 的机会都没有。

```C++
int Compute() {
    return 42;   // 辛苦算出来的结果
}

std::thread t(Compute);   // 42 去哪了？丢了
t.join();
```

以前要拿结果，只能靠共享变量加锁，写起来又脏又容易错。promise 和 future 就是为这个痛点准备的：**一条一次性的传话通道，一头塞值（promise），一头取值（future）**。

**基本用法**

```C++
#include <future>
#include <thread>
#include <iostream>

int main() {
    std::promise<int> p;
    std::future<int> fut = p.get_future();

    std::thread producer([&p] {
        int result = 42;   // 假装是复杂计算
        p.set_value(result);
    });

    std::cout << "got " << fut.get() << '\n';   // get() 阻塞，直到 set_value
    producer.join();
}
// 输出：got 42
```

`fut.get()` 会阻塞直到 promise 那头 set 值，之后值直接返回。注意这条通道是**一次性**的：set 一次、get 一次，set 第二次会抛 `future_error`。要反复传值就得自己搭队列，或者直接上第三方的 channel 实现。

**异常也能传——这才是真正值钱的能力**

```C++
std::promise<int> p;
std::future<int> fut = p.get_future();

std::thread producer([&p] {
    try {
        throw std::runtime_error("something broke");
    } catch (...) {
        p.set_exception(std::current_exception());   // 原样转交
    }
});

try {
    fut.get();   // 异常在这里重新抛出
} catch (const std::exception& e) {
    std::cout << "caught: " << e.what() << '\n';
}
producer.join();
// 输出：caught: something broke
```

线程里的异常被搬到调用线程重新抛出，"子线程炸了没人知道"变成了"主线程正常 catch"，排查问题的体验完全是两个世界。

**packaged_task：懒得手写 set 的就用它**

每次都手写 set_value / set_exception 太啰嗦，`std::packaged_task` 把这层样板代码包掉了——函数正常返回就自动 set_value，抛异常就自动 set_exception：

```C++
int main() {
    std::packaged_task<int(int, int)> task([](int a, int b) { return a + b; });
    std::future<int> fut = task.get_future();

    std::thread t(std::move(task), 20, 22);   // task 只能移动
    t.join();

    std::cout << fut.get() << '\n';
}
// 输出：42
```

可以把它理解成"包了 promise 的函数包装器"，拿现成函数/lambda 起线程收结果的场景比裸 promise 顺手。

**async：一行搞定**

不想自己管 thread 对象的话，`std::async` 是最省事的：

```C++
int main() {
    auto fut = std::async(std::launch::async, [](int a, int b) {
        return a + b;
    }, 20, 22);

    std::cout << fut.get() << '\n';
}
// 输出：42
```

两个坑要交代：

1. **明确写 `std::launch::async`**。不写的话默认策略是 `async | deferred`，编译器可以偷懒选择 deferred——不开新线程，拖到你调 get() 时才在当前线程同步执行。想真并行就必须显式指定。
2. **async 返回的 future 析构时会阻塞等结果**。这是标准里专门为它规定的特殊行为（别的 future 析构不管），临时忘了 get，析构一样会等，一不留神代码就被意外串行化了。

最后给个选择思路：简单的一次性任务用 `std::async`；要自己控制线程的生命周期就用 `thread + packaged_task`；要在线程之间设计通信协议（传值、传异常、发完成信号）才需要直接上手 `promise/future`。多数业务代码停在 async 这一层就够了，真的碰到线程池、高频任务调度的场景，再往深处走。
