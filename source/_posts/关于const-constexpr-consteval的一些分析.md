---
title: C++中的 const/constexpr/consteval/constinit 的一些分析
date: 2026-08-12 15:30:12
categories: [C++]
tags:
  - C++
  - 编程语言
---

## 引言

C++ 中有很多关于常量性的概念，后来 C++20 新特性又添加了关于常量概念的新特性，这些关于常量性的关键字有着不同的用法和特性，虽然项目中大量使用但是平时没有对其进行深入剖析。现在本文尝试对这些不同的关键字进行汇总并尝试进行深入理解，以期能了解 C++ 引入这些不同常量性的真实设计意图。

## const

### 修饰变量

一般的，也是最常见的用法是使用 const 修饰变量，表示这个变量是常量，不可被修改。因为变量一旦创建后就不能被修改，所以 const 变量必须初始化。

值得注意的是 const 变量默认是内部链接的，所以可以直接放到头文件中进行定义，每个 cpp #include 该头文件不会报变量重定义的错误，但是每个模块会自动生成该变量的一个副本。C++ 之所以 让全局 `const` 默认内部链接，**首要目的就是允许在头文件直接定义常量，多文件 include 不会触发链接重定义错误**，同时减少全局符号污染，方便编译期常量优化。

#### 修饰引用与指针

```cpp
int i = 10;
const int &a = i;  // 底层const, 法通过 a 修改 i 的值
```

因为指针本身是一个对象，指针本身指向的值和自身是否是 const 是两个问题。所以用顶层 const 表示指针本身是一个常量，而用底层 const 表示指针指向的值是一个常量。

```cpp
int i = 10;
const int *ptr = &i;   // 底层 const：指针指向的值是常量，不能通过 ptr 修改 i
int const *ptr2 = &i;  // 等价于上面的写法
int *const ptr3 = &i;  // 顶层 const：指针本身是常量，不能修改 ptr3 指针本身
```

### 尽量使用 const 来代替 #define

在 C 语言的时代，定义常量只有一种朴素的办法 —— 预处理宏：

```cpp
#define MAX_SIZE 1024
#define PI       3.14159
#define GREETING "hello"
```

能跑，但代价不小。宏在**预处理阶段**就被处理成纯文本替换：预处理器不认类型、不认作用域、不认 C++ 语义，它只负责把 `MAX_SIZE` 这几个字符换成 `1024`。替换发生在编译器真正开始工作之前，于是本该由语言机制提供的保障 `类型检查、作用域、调试信息` 在这批常量上全部失效。

C++ 里应当改用 `const`来定义它们：

```cpp
const int    MaxSize  = 1024;
const double Pi       = 3.14159;
const char*  const Greeting = "hello";   // 或 constexpr std::string_view
```

这样做有几个好处：

#### 1. 目标代码一般更小

宏的语义是**每个使用点都展开一份**，所以预处理器会把所有 `MAX_SIZE` 都替换成 `1024`，可能导致目标码出现多份 `1024` 字面量，从而导致目标代码体积增大。


#### 2. 编译错误信息更直观

先看宏的报错现场：

```cpp
#define MAX_SIZE 1024
int main() {
    MAX_SIZE = 2048;   // 想改"常量"
    return 0;
}
```

编译器看到的是 `1024 = 2048;`，于是报出的是"给一个字面量赋值"这种驴唇不对马嘴的错误（GCC 会说 `lvalue required as left operand of assignment`，Clang 还要额外给你一个 `note: expanded from macro 'MAX_SIZE'` 你才能勉强猜出是谁惹的祸）。

换成 `const` 后：

```cpp
const int MaxSize = 1024;
MaxSize = 2048;   // error: assignment of read-only variable 'MaxSize'
```

符号名被完整保留，报错直接点名 `MaxSize`，类型不匹配等问题也能给出精确的期望 / 实际类型。


#### 3. 类型安全，且能参与语言机制

宏没有类型，它只是文本。`#define PI 3.14159` 的类型是 `double`，你无法让它在上文中当作 `float` 使用，也无法让编译器替你检查。


#### 4. 作用域受控，不污染全局命名空间

`#define` 从定义处起一直生效到 `#undef` 或文件结束，作用于**所有**后续文本，而且不属于任何命名空间、类或函数：

```cpp
#define version 1   // 从此任何叫 version 的标识符、成员、变量都可能被悄悄改写
```

`const` 遵循正常的 C++ 作用域规则：可以定义在命名空间里、类里、函数里，不同作用域可以重名，也可以被 `using` 引入。再加上全局 `const` 默认内部链接（见前文），它还可以安全地直接写在头文件中，多个 `.cpp` 同时 `#include` 也不会触发重定义错误。

#### 5. 可调试

宏在调试信息里**不存在**：你没法给 `MAX_SIZE` 设断点，watch 窗口里也看不到它，看到的只有展开后的字面量。`const` 变量是真实符号，可以在调试器中查看、求值、打印，配合 `-g` / `/Zi` 能直接定位。

#### 6. 避开宏的经典陷阱

函数式宏尤其危险：

```cpp
#define MAX(a, b) ((a) > (b) ? (a) : (b))
int i = 0, j = 0;
int m = MAX(i++, j++);   // i 和 j 各被自增了两次！
```

参数会被**重复求值**（上面这段代码里 `i++`、`j++` 各执行两遍）；运算符优先级要求每个参数、整个表达式都必须加括号，漏一个括号就是一个隐藏 bug；并且它无法被调试、无法取地址。C++ 中对应的替代品是 `inline` 函数 / `constexpr` 函数 / 函数模板，它们拥有真实的参数语义：求值一次、有类型检查、有正常作用域、可调试、可内联。


### 修饰函数

C++中，允许对函数进行const修饰，表示这个函数是常量函数，不可以修改类的成员变量。C++对const函数常量性的定义是bitwise的，即函数内部的所有操作都不会修改对象的bit位。 const 是编译器层面的 const 检查：只要对象内存里的每一个 bit 都不被修改，就认为这个成员函数是 const 的；不关心业务逻辑上对象的语义有没有变，不关心对象指针指向的外部数据是否被改变。

mutable 关键字可以在 const 成员函数内修改成员变量，来实现logical constness 用来弥补 bitwise const 的不足。

如果一个类中有同名的const 成员函数和非const版本，而且代码逻辑基本相同，那么可以让非const版本调用const版本来消除一定的重复，但是反过来不行。这种技术是值得了解的：

```cpp
class TextBlock {
public:
    explicit TextBlock(std::string t) : text_(std::move(t)) {}

    // const 版本：承载全部真实逻辑（越界检查、加锁、日志……）
    const char& operator[](std::size_t pos) const {
        // ... 边界检查 / 加锁 / 访问日志 ...
        return text_[pos];
    }

    // 非 const 版本：复用 const 版本，两个转型各司其职
    char& operator[](std::size_t pos) {
        return const_cast<char&>(                      // 2. 去掉返回值的 const
            static_cast<const TextBlock&>(*this)[pos]  // 1. 加上 const 以调用 const 版本
        );
    }

private:
    std::string text_;
};
```


## constexpr

constexpr 修饰的变量可以在编译期进行求值，编译器会来验证这个变量的值是否是常量表达式，如果是的话则会在编译期进行求值提高程序性能。

```cpp
constexpr int a = 10;
```

constexpr 也可以修饰函数，表示这个函数是常量表达式函数，可以用于编译期求值，如果编译期无法求值则退化为普通函数。`constexpr` 标记的东西：**具备编译期求值的能力，但不强制**

```cpp
constexpr int add(int a, int b) {
    return a + b;
}

int value = add(10, 20); // 编译期求值，value = 30
```

## consteval

consteval 是C++20新增关键字，`constexpr` 是 “可选编译期”，而很多场景我们想要**强制编译期求值**，绝对不能落到运行期，防止某些函数意外在运行期执行带来性能隐患，于是 C++20 新增 `consteval`。

`consteval` 修饰函数，代表**立即函数（immediate function）**，这个函数只能在编译期上下文调用，任何运行期调用直接编译报错。

```cpp
consteval int square(int x) {
    return x * x;
}

int main() {
    constexpr int s1 = square(5);//编译期调用
    int v = 10;
    int s2 = square(v);//报错，不能运行期调用
    return 0;
}
```

## constinit

引入constinit 是为了解决经典 C++ 坑：**静态初始化顺序灾难（Static Initialization Order Fiasco, SIOF）**。C++所有全局变量、static 局部变量、static 类成员，都属于**静态存储期**。它们的初始化分为两个阶段：

1. **静态初始化（static initialization）**
   - 包括零初始化 + 常量初始化
   - 在程序启动、进入 `main()` **之前**完成，由编译器 / 加载器直接把值写入内存。
2. **动态初始化（dynamic initialization）**
   - 运行期初始化，在静态初始化完成之后、`main()` 之前执行。
   - 同一翻译单元内：按代码从上到下顺序初始化。
   - 不同翻译单元之间：初始化顺序完全未定义。
constinit修饰**静态存储期变量**，要求变量的初始化表达式必须是**常量表达式**，它会保证变量走**静态初始化**。如果表达式不满足，编译直接报错。

```cpp
// file1.cpp
#include <iostream>
extern int b;
int a = b + 1; // 动态初始化！跨文件顺序不确定，SIOF风险

// file2.cpp
int b = 42; // 动态初始化！
```
使用constinit 来规定变量b在常量初始哈阶段完成，这样file1.cpp使用b就是安全的了

```cpp
// file1.cpp
#include <iostream>
extern int b;
int a = b + 1; // 因为b提前初始化了，这里b的值一定正确

// file2.cpp
constinit int b = 42; // 静态初始化（常量初始化），在main之前b的值就是42了
```

### 非常量表达式的，定义在不同编译单元的non-local static对象的初始化顺序无法保证的问题

如果变量初始化**必须是运行期表达式**，又要控制初始化顺序（规避 SIOF），这时候用constinit是无法解决的，常见工程方案：把全局变量包装成函数内 static（首次调用才初始化）

```cpp
// file2.pp
extern FileSystem tfs;
Directory::Directory(params)
{
    std::size_t = tfs.numDisks();//不知道什么时候tfs会被初始化
}

//file1.cpp
extern FileSystem tfs;
FileSystem tfs;
```

解决方案：

```cpp
FileSystem& tfs() {
    static FileSystem fs;//C++保证函数内的local static对象会在该函数被调用期间，首次遇到该对象定义式时被初始化。
                        //C++11保证local static初始化期间，线程安全，语言层面保证。
    return fs;
}

Directory::Directory(params)
{
    std::size_t = tfs().numDisks();//调用tfs函数，tfs函数内部的local static对象fs在函数调用期间被初始化
}
```

