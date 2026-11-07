---
title: CMake构建工具Tutorial
date: 2026-07-18 12:53:36
categories: [构建工具]
tags:
  - CMake
  - 构建工具
---

CMake 是目前 C/C++ 生态中最主流的跨平台构建工具。无论在 Linux 上用 GCC、Windows 上用 MSVC，还是 macOS 上用 Clang，CMake 都能生成对应的构建文件。这篇文章从零开始，介绍 CMake 的核心用法。

<!-- more -->

## 1. CMake 概述

### 什么是 CMake？

CMake 是一个**跨平台的项目构建工具**。它本身并不直接编译代码，而是根据你编写的 `CMakeLists.txt` 配置文件，**自动生成本地化的构建文件**（如 Makefile、Visual Studio 工程文件等），然后交给底层编译器去完成实际的编译工作。

在 CMake 出现之前，开发者通常需要手写 Makefile 来管理编译流程。但手写 Makefile 存在几个明显的问题：

- **平台依赖性强**：不同操作系统的 Makefile 语法和工具链差异很大
- **工作量大**：项目规模稍大，Makefile 就会变得极其冗长
- **容易出错**：手动维护文件之间的依赖关系，稍有不慎就会遗漏

CMake 就是为了解决这些问题而生的。可以把它理解成 **"Makefile 的自动生成器"**：开发者只描述编译规则，CMake 把这些规则翻译成目标平台上的构建脚本。

### CMake 的优势

| 优势 | 说明 |
|------|------|
| 跨平台 | 一套配置，支持 Linux / Windows / macOS 等多个平台 |
| 管理大型项目 | 支持多目录、多模块、多子项目的组织方式 |
| 简化构建流程 | 自动处理依赖关系、编译顺序等复杂问题 |
| 可扩展性强 | 支持自定义模块和函数，灵活适配各种需求 |

---

## 2. CMake 的使用

CMake 的命令**不区分大小写**，`cmake_minimum_required`、`CMAKE_MINIMUM_REQUIRED`、`Cmake_Minimum_Required` 都是合法写法。不过为了代码风格统一，建议团队内保持一致的命名习惯。

### 2.1 注释

CMake 支持两种注释方式。

**行注释**使用 `#`，可以出现在任意位置：

```cmake
# 这是一行注释
cmake_minimum_required(VERSION 3.0.0)   # 行尾也可以加注释
```

**块注释**使用 `#[[ ]]`，适合多行说明：

```cmake
#[[
这是一个 CMakeLists.txt 文件。
这里可以写多行注释，
用于描述项目的整体信息。
]]#
cmake_minimum_required(VERSION 3.0.0)
```

### 2.2 只有源文件的项目

最简单的 CMake 项目，就是把若干个源文件编译成一个可执行程序。我们以一个 **"数学工具库"** 为例来演示。

#### 2.2.1 共处一室

先看最朴素的场景：所有源文件都放在同一个目录下。

```
$ tree
.
├── gcd.c
├── factorial.c
├── fibonacci.c
├── is_even.c
├── mathlib.h
└── main.c
```

各文件的职责：

- `is_even.c`：实现 `int is_even(int n)`，判断一个整数是否为偶数
- `factorial.c`：实现 `long factorial(int n)`，计算 n 的阶乘
- `fibonacci.c`：实现 `long fibonacci(int n)`，返回第 n 个斐波那契数
- `gcd.c`：实现 `int gcd(int a, int b)`，计算两个整数的最大公约数
- `mathlib.h`：头文件，声明以上四个函数
- `main.c`：测试主程序，调用各函数并输出结果

**CMakeLists.txt（3 条核心命令）：**

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
add_executable(demo gcd.c factorial.c fibonacci.c is_even.c main.c)
```

三条命令的作用分别是：

**1. `cmake_minimum_required`**：指定 CMake 的最低版本要求。这条命令是**可选的**，但如果省略，CMake 在处理较新语法时可能会给出版本过低的警告，建议写上。

**2. `project`**：定义工程名称。完整语法如下：

```cmake
project(<PROJECT-NAME> [<language-name>...])
project(<PROJECT-NAME>
       [VERSION <major>[.<minor>[.<patch>[.<tweak>]]]]
       [DESCRIPTION <project-description-string>]
       [HOMEPAGE_URL <url-string>]
       [LANGUAGES <language-name>...])
```

- `PROJECT-NAME`：工程名称（必填）
- `VERSION`：工程版本号（可选）
- `DESCRIPTION`：工程描述（可选）
- `HOMEPAGE_URL`：项目主页 URL（可选）
- `LANGUAGES`：支持的编程语言（可选，默认支持所有语言）

**3. `add_executable`**：定义要生成的可执行程序。语法：

```cmake
add_executable(可执行程序名 源文件1 [源文件2] ...)
```

注意：**可执行程序名与 `project` 中的项目名没有任何关系**，两者可以完全不同。多个源文件之间用**空格**分隔：

```cmake
add_executable(demo gcd.c factorial.c fibonacci.c is_even.c main.c)
```

**执行构建：**

```shell
# cmake 命令原型：cmake + CMakeLists.txt 所在路径
$ cmake .
```

执行完成后，目录下会多出 `CMakeCache.txt`、`CMakeFiles/`、`cmake_install.cmake`、`Makefile` 等文件。接着执行 `make`：

```shell
$ make
Scanning dependencies of target demo
[ 20%] Building C object CMakeFiles/demo.dir/gcd.c.o
[ 40%] Building C object CMakeFiles/demo.dir/factorial.c.o
[ 60%] Building C object CMakeFiles/demo.dir/fibonacci.c.o
[ 80%] Building C object CMakeFiles/demo.dir/is_even.c.o
[100%] Building C object CMakeFiles/demo.dir/main.c.o
[100%] Linking C executable demo
[100%] Built target demo
```

最终生成的可执行程序名为 `demo`（即 `add_executable` 中指定的名字）。

#### 2.2.2 VIP 包房

上面的做法虽然简单，但有个问题：在源码目录下直接执行 `cmake`，会生成一大堆中间文件，把源码目录搞得杂乱无章。更好的做法是**单独创建一个 `build` 目录**来存放构建产物：

```shell
$ mkdir build
$ cd build
$ cmake ..
```

注意路径是 `..`（两个点），因为 `CMakeLists.txt` 在 `build` 目录的上一级。

执行后，`CMakeCache.txt`、`CMakeFiles/`、`Makefile` 等文件都生成在 `build` 目录中。后续执行 `make` 产生的目标文件和可执行程序也会在 `build` 里。这样就实现了**源文件与构建产物的完全隔离**，源码目录保持干净整洁。

### 2.3 私人订制

CMake 提供了丰富的配置选项，可以根据需要定制构建行为。

#### 2.3.1 定义变量

当源文件较多时，在 `add_executable` 里逐个罗列文件名会显得很冗长。可以用 `set` 命令定义变量来简化：

```cmake
# set 命令语法
# 方括号中的参数为可选项
set(<variable> <value>...)
```

用变量来收集源文件列表：

```cmake
set(SRC_LIST gcd.c factorial.c fibonacci.c is_even.c main.c)

add_executable(demo ${SRC_LIST})
```

`${SRC_LIST}` 是引用变量的方式，会在展开时替换为变量的值。

#### 2.3.2 指定使用的 C++ 标准

在命令行编译时，我们用 `-std=c++11` 来指定 C++ 标准。在 CMake 中，对应的配置项是 `CMAKE_CXX_STANDARD`，有两种设置方式：

**方式 1：在 CMakeLists.txt 中用 `set` 指定**

```cmake
# 使用 C++11 标准
set(CMAKE_CXX_STANDARD 11)
# 使用 C++14 标准
set(CMAKE_CXX_STANDARD 14)
# 使用 C++17 标准
set(CMAKE_CXX_STANDARD 17)
```

**方式 2：在 cmake 命令行中指定**

```shell
# 使用 C++11 标准
cmake .. -DCMAKE_CXX_STANDARD=11
# 使用 C++14 标准
cmake .. -DCMAKE_CXX_STANDARD=14
# 使用 C++17 标准
cmake .. -DCMAKE_CXX_STANDARD=17
```

> 命令行方式的 `-D` 参数用于向 CMake 传递变量定义，优先级高于 CMakeLists.txt 中的 `set`。

#### 2.3.3 指定输出的路径

默认情况下，生成的可执行程序就在构建目录（如 `build/`）中。如果想把它放到指定位置，可以通过 `EXECUTABLE_OUTPUT_PATH` 来控制：

```cmake
set(HOME ${PROJECT_SOURCE_DIR}/bin)
set(EXECUTABLE_OUTPUT_PATH ${HOME})
```

第一行定义了一个指向 `bin` 目录的变量，第二行将其设置为可执行程序的输出路径。**如果路径中的子目录不存在，CMake 会自动创建**。

> 如果使用相对路径（如 `./output`），则 `./` 对应的是 **Makefile 所在目录**（即构建目录），而非源码目录。

### 2.4 搜索文件

当项目文件较多时，手动把每个源文件名写进 CMakeLists.txt 既繁琐又容易遗漏。CMake 提供了自动搜索源文件的机制。

#### 2.4.1 `aux_source_directory` 命令

```cmake
aux_source_directory(<dir> <variable>)
```

- `dir`：要搜索的目录路径
- `variable`：用于存储搜索到的源文件列表的变量名

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
include_directories(${PROJECT_SOURCE_DIR}/include)
# 搜索 src 目录下的所有源文件
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src SRC_LIST)
add_executable(demo ${SRC_LIST})
```

这条命令会把 `src/` 目录下的所有源文件收集到 `SRC_LIST` 变量中。

#### 2.4.2 `file` 命令

相比 `aux_source_directory`，`file` 命令更加灵活：

```cmake
file(GLOB <variable> <globbing_expression>...)
file(GLOB_RECURSE <variable> <globbing_expression>...)
```

- `GLOB`：在指定目录中搜索匹配的文件，生成列表存入变量
- `GLOB_RECURSE`：递归搜索所有子目录

```cmake
# 搜索 src 目录下所有 .cpp 文件
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
# 搜索 include 目录下所有 .h 文件
file(GLOB HEAD_LIST ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h)
```

其中 `CMAKE_CURRENT_SOURCE_DIR` 表示**当前 CMakeLists.txt 文件所在的目录**。路径可以用双引号包裹，也可以不加。

> **提示**：`file(GLOB)` 在 CMake 运行时搜索文件。如果你在构建之后新增了源文件，需要重新执行 `cmake` 才能被识别到。从 CMake 3.12 起，还可以使用 `CONFIGURE_DEPENDS` 选项让文件变化自动触发重新配置。

### 2.5 包含头文件

当源文件和头文件分属不同目录时，编译器默认是找不到头文件的。需要用 `include_directories` 来指定头文件搜索路径：

```cmake
include_directories(<headpath>)
```

来看一个典型项目的目录结构：

```
.
├── build
├── CMakeLists.txt
├── include
│   └── mathlib.h
└── src
    ├── factorial.c
    ├── fibonacci.c
    ├── gcd.c
    ├── is_even.c
    └── main.c
```

完整的 CMakeLists.txt 如下：

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
set(CMAKE_CXX_STANDARD 11)
# 设置可执行程序输出路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
# 包含头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)
# 搜索源文件
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
add_executable(demo ${SRC_LIST})
```

其中 `PROJECT_SOURCE_DIR` 表示**执行 cmake 命令时后面紧跟的目录**，一般是工程的根目录。

### 2.6 制作动态库或静态库

在实际开发中，我们经常需要把通用的代码封装成库，供其他项目复用。CMake 中用 `add_library` 来创建库文件。

#### 2.6.1 制作静态库

```cmake
add_library(<库名称> STATIC <源文件1> [<源文件2>] ...)
```

- Linux 下静态库的命名规则是 `lib` + 库名 + `.a`，使用时只需指定库名，前后缀由 CMake 自动补全
- Windows 下同样只需指定库名

示例项目结构（`include/mathlib.h`、`src/` 下四个 `.c` 文件）：

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
include_directories(${PROJECT_SOURCE_DIR}/include)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
add_library(mathlib STATIC ${SRC_LIST})
```

构建后会生成静态库 `libmathlib.a`（Linux）。

#### 2.6.2 制作动态库

```cmake
add_library(<库名称> SHARED <源文件1> [<源文件2>] ...)
```

- Linux 下动态库的命名规则是 `lib` + 库名 + `.so`

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
include_directories(${PROJECT_SOURCE_DIR}/include)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
add_library(mathlib SHARED ${SRC_LIST})
```

构建后会生成动态库 `libmathlib.so`（Linux）。

#### 2.6.3 指定库的输出路径

与可执行程序类似，库文件也可以指定输出位置，有以下两种方式：

**方式 1：使用 `EXECUTABLE_OUTPUT_PATH`（适用于动态库）**

Linux 下动态库默认具有执行权限，因此可以用 `EXECUTABLE_OUTPUT_PATH` 来指定路径：

```cmake
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
add_library(mathlib SHARED ${SRC_LIST})
```

**方式 2：使用 `LIBRARY_OUTPUT_PATH`（静态库和动态库均适用）**

静态库没有执行权限，所以方式 1 对静态库无效。`LIBRARY_OUTPUT_PATH` 则是通用的：

```cmake
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
# 生成动态库
add_library(mathlib SHARED ${SRC_LIST})
# 或生成静态库
# add_library(mathlib STATIC ${SRC_LIST})
```

### 2.7 包含库文件

库制作完成后，其他项目就可以链接并使用它了。下面分别介绍静态库和动态库的链接方式。

#### 2.7.1 链接静态库

假设我们的项目结构如下：

```
.
├── build
├── CMakeLists.txt
├── include
│   └── mathlib.h
├── lib
│   └── libmathlib.a     # 制作好的静态库
└── src
    └── main.c
```

CMake 中用 `link_libraries` 来链接静态库：

```cmake
link_libraries(<static lib> [<static lib>...])
```

这是一条**全局设置命令**，对其后定义的所有目标都会生效。参数可以是库的全名（如 `libmathlib.a`）或简写（如 `mathlib`）。

如果库文件不在系统默认搜索路径中，需要用 `link_directories` 指定其所在目录：

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
# 搜索源文件
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
# 包含头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)
# 指定静态库所在路径
link_directories(${PROJECT_SOURCE_DIR}/lib)
# 链接静态库
link_libraries(mathlib)
add_executable(demo ${SRC_LIST})
```

#### 2.7.2 链接动态库

对于动态库，推荐使用更精确的 `target_link_libraries` 命令：

```cmake
target_link_libraries(
    <target>
    <PRIVATE|PUBLIC|INTERFACE> <item>...
    [<PRIVATE|PUBLIC|INTERFACE> <item>...]...)
```

- `target`：要链接库的目标（可以是可执行文件、静态库或动态库）
- `PRIVATE | PUBLIC | INTERFACE`：链接的可见性控制，默认为 `PUBLIC`

三者的区别在于**链接的传递性**。动态库的链接具有传递性：如果 A 链接了 B 和 C，D 又链接了 A，那么 D 也会间接链接到 B 和 C：

```cmake
target_link_libraries(A B C)
target_link_libraries(D A)   # D 相当于也链接了 B 和 C
```

| 关键字 | 效果 |
|--------|------|
| `PUBLIC` | 库被链接到 target，且其符号导出给第三方 |
| `PRIVATE` | 库仅链接到 target，不传递给第三方 |
| `INTERFACE` | 库不链接到 target，仅导出符号给第三方 |

**静态库与动态库的加载时机**是一个重要区别：

- **静态库**在**链接阶段**就被打包进可执行程序，程序启动时即加载完毕
- **动态库**在链接阶段仅建立引用关系，**运行时**调用到才真正加载

因此，**动态库的链接命令应写在 `add_executable` 之后**。

**链接系统动态库示例**（如线程库）：

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/*.c)
# 生成可执行程序
add_executable(demo ${SRC_LIST})
# 链接系统线程库（libpthread.so -> pthread）
target_link_libraries(demo pthread)
```

`pthread` 是 `libpthread.so` 的"掐头去尾"写法，即去掉 `lib` 前缀和 `.so` 后缀。

**链接第三方动态库示例**：

项目结构：`include/mathlib.h`、`lib/libmathlib.so`、`src/main.c`

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
include_directories(${PROJECT_SOURCE_DIR}/include)
add_executable(demo ${SRC_LIST})
target_link_libraries(demo pthread mathlib)
```

如果运行时报错 `error while loading shared libraries: libmathlib.so: cannot open shared object file`，说明运行时找不到动态库的位置。解决方法是在 `add_executable` 之前用 `link_directories` 指定库路径：

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
# 指定头文件路径
include_directories(${PROJECT_SOURCE_DIR}/include)
# 指定动态库路径
link_directories(${PROJECT_SOURCE_DIR}/lib)
# 生成可执行程序
add_executable(demo ${SRC_LIST})
# 链接动态库
target_link_libraries(demo pthread mathlib)
```

#### 2.7.3 `target_link_libraries` 与 `link_libraries` 对比

| | `target_link_libraries` | `link_libraries` |
|---|---|---|
| 作用范围 | 指定**某个目标**链接哪些库 | 设置**全局**链接库，对之后所有目标生效 |
| 语法 | `target_link_libraries(target lib1 lib2 ...)` | `link_libraries(lib1 lib2 ...)` |
| 优点 | 精确控制、支持调试/发布版本区分、灵活易维护 | 写法简单 |
| 缺点 | 语法稍复杂 | 缺乏目标级控制、容易产生意外的全局依赖 |
| 适用场景 | 中大型项目、库之间有复杂依赖关系 | 简单项目或临时快速设置 |

**建议优先使用 `target_link_libraries`**，它能提供更精确的控制。`link_libraries` 更适合简单项目或临时测试场景。

### 2.8 日志

在编写 CMakeLists.txt 时，适当输出日志有助于调试。CMake 的 `message` 命令支持多种日志级别：

```cmake
message([<mode>] "message to display" ...)
```

| 模式 | 说明 |
|------|------|
| （无） | 输出重要消息 |
| `STATUS` | 输出非重要消息（显示在 stdout） |
| `WARNING` | 输出 CMake 警告，继续执行 |
| `AUTHOR_WARNING` | 输出 CMake 开发者警告，继续执行 |
| `SEND_ERROR` | 输出 CMake 错误，跳过生成步骤但继续处理 |
| `FATAL_ERROR` | 输出 CMake 致命错误，立即终止所有处理 |

除 `STATUS` 输出到 stdout 外，其余均输出到 stderr。示例：

```cmake
# 输出一般信息
message(STATUS "Source path: ${PROJECT_SOURCE_DIR}")
# 输出警告信息
message(WARNING "This is a warning message")
# 输出致命错误
message(FATAL_ERROR "Something went wrong!")
```

### 2.9 变量操作

CMake 中的变量和列表操作是编写灵活构建脚本的基础。

#### 2.9.1 追加

**使用 `set` 追加**：本质是拼接后覆盖原值

```cmake
set(变量名1 ${变量名1} ${变量名2} ...)
```

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
set(GREETING "hello, world")
file(GLOB SRC_1 ${PROJECT_SOURCE_DIR}/src1/*.c)
file(GLOB SRC_2 ${PROJECT_SOURCE_DIR}/src2/*.c)
# 将 SRC_2 和 GREETING 追加到 SRC_1
set(SRC_1 ${SRC_1} ${SRC_2} ${GREETING})
message(STATUS "SRC_1: ${SRC_1}")
```

**使用 `list(APPEND)` 追加**：更推荐的方式

```cmake
list(APPEND <list> [<element> ...])
```

```cmake
list(APPEND SRC_1 ${SRC_2} ${GREETING})
```

> **小知识**：CMake 中的 `set` 可以创建列表（list），列表本质上是以**分号 `;` 分隔**的字符串。例如 `set(var a b c d e)` 创建的列表等价于 `a;b;c;d;e`。
>
> ```cmake
> set(tmp1 a;b;c;d;e)
> set(tmp2 a b c d e)
> message(${tmp1})    # 输出: abcde
> message(${tmp2})    # 输出: abcde
> ```
>
> `tmp1` 和 `tmp2` 的结果完全相同。

#### 2.9.2 移除元素

一个典型场景：搜索目录时找到了所有源文件，但需要剔除测试文件 `test_main.c` 后再制作库。

```cmake
list(REMOVE_ITEM <list> <value> [<value> ...])
```

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/*.c)
# 移除前
message(STATUS "Before: ${SRC_LIST}")
# 移除 test_main.c
list(REMOVE_ITEM SRC_LIST ${PROJECT_SOURCE_DIR}/test_main.c)
# 移除后
message(STATUS "After: ${SRC_LIST}")
```

> **注意**：`file(GLOB)` 搜索得到的是**绝对路径**，列表中每个元素都是完整的绝对路径。因此 `list(REMOVE_ITEM)` 也必须使用绝对路径来匹配，否则不会生效。

**`list` 命令的完整子命令一览**：

| 子命令 | 说明 |
|--------|------|
| `list(LENGTH <list> <output>)` | 获取列表长度 |
| `list(GET <list> <index> <output>)` | 按索引读取元素（索引从 0 开始，-1 表示最后一个） |
| `list(JOIN <list> <glue> <output>)` | 用连接符合并为字符串 |
| `list(FIND <list> <value> <output>)` | 查找元素，返回索引，未找到返回 -1 |
| `list(APPEND <list> [<element>...])` | 追加元素 |
| `list(INSERT <list> <index> <element>...)` | 在指定位置插入 |
| `list(PREPEND <list> [<element>...])` | 插入到列表开头 |
| `list(POP_BACK <list> [<out-var>...])` | 移除最后一个元素 |
| `list(POP_FRONT <list> [<out-var>...])` | 移除第一个元素 |
| `list(REMOVE_ITEM <list> <value>...)` | 移除指定元素 |
| `list(REMOVE_AT <list> <index>...)` | 移除指定索引的元素 |
| `list(REMOVE_DUPLICATES <list>)` | 移除重复元素 |
| `list(REVERSE <list>)` | 列表翻转 |
| `list(SORT <list> [...])` | 列表排序 |

`list(SORT)` 还支持额外的排序选项：

- `COMPARE`：`STRING`（字母序，默认）、`FILE_BASENAME`（按文件名排序）、`NATURAL`（自然数序）
- `CASE`：`SENSITIVE`（默认）、`INSENSITIVE`
- `ORDER`：`ASCENDING`（默认）、`DESCENDING`

### 2.10 宏定义

在 C/C++ 代码中，我们经常用宏来控制某段代码是否参与编译：

```c
#include <stdio.h>

#define MAX_COUNT 5

int main()
{
    int i;
#ifdef DEBUG
    printf("Debug mode is enabled\n");
#endif
    for (i = 0; i < MAX_COUNT; ++i)
    {
        printf("Hello, CMake!\n");
    }
    return 0;
}
```

在命令行中，我们用 `gcc test.c -DDEBUG -o app` 来定义 `DEBUG` 宏。CMake 中对应的方式是 `add_definitions`：

```cmake
add_definitions(-D<宏名称>)
```

在 CMakeLists.txt 中使用：

```cmake
cmake_minimum_required(VERSION 3.0)
project(TEST)
# 定义 DEBUG 宏
add_definitions(-DDEBUG)
add_executable(demo ./test.c)
```

这样构建时就会自动加上 `-DDEBUG` 编译选项，代码中 `#ifdef DEBUG` 的部分就会被编译。

---

## 3. 预定义宏速查表

CMake 内置了许多预定义变量（宏），在编写构建脚本时非常实用：

| 宏 | 说明 |
|---|---|
| `PROJECT_SOURCE_DIR` | 执行 cmake 命令时紧跟的目录，一般是工程根目录 |
| `PROJECT_BINARY_DIR` | 执行 cmake 命令的目录（构建目录） |
| `CMAKE_CURRENT_SOURCE_DIR` | 当前正在处理的 CMakeLists.txt 所在目录 |
| `CMAKE_CURRENT_BINARY_DIR` | 当前 target 的编译输出目录 |
| `EXECUTABLE_OUTPUT_PATH` | 自定义可执行程序的输出路径 |
| `LIBRARY_OUTPUT_PATH` | 自定义库文件的输出路径 |
| `PROJECT_NAME` | `project()` 命令定义的项目名称 |
| `CMAKE_BINARY_DIR` | 项目的实际构建根路径（如 `build/` 目录） |

---

## 4. 嵌套的 CMake

前面的例子都只有一个 `CMakeLists.txt`，适合小型项目。但在实际开发中，项目往往包含多个模块、多个源码目录。如果所有配置都塞进一个文件，很快就会变得臃肿难维护。

CMake 的解决方案是：**给每个源码目录都放一个独立的 `CMakeLists.txt`**（头文件目录不需要），由顶层文件统一调度。这样每个文件都不复杂，模块之间职责清晰，改起来也更灵活。

来看一个具体的项目结构：

```
$ tree
.
├── build
├── CMakeLists.txt
├── include
│   ├── mathlib.h
│   └── strlib.h
├── mathlib
│   ├── CMakeLists.txt
│   ├── factorial.c
│   ├── fibonacci.c
│   ├── gcd.c
│   └── is_even.c
├── strlib
│   ├── CMakeLists.txt
│   ├── str_reverse.c
│   └── str_upper.c
├── test1
│   ├── CMakeLists.txt
│   └── math_test.c
└── test2
    ├── CMakeLists.txt
    └── str_test.c
```

各目录的职责如下：

- `include/`：公共头文件，`mathlib.h` 声明数学函数，`strlib.h` 声明字符串函数
- `mathlib/`：数学工具模块，包含阶乘、斐波那契、最大公约数、奇偶判断
- `strlib/`：字符串工具模块，包含字符串反转、转大写
- `test1/`：数学模块的测试程序
- `test2/`：字符串模块的测试程序

### 4.1 准备工作

#### 4.1.1 节点关系

嵌套的 `CMakeLists.txt` 构成一棵**树状结构**，最顶层的是根节点，其余是子节点。变量的作用域遵循以下规则：

- 根节点中的变量**全局有效**，所有子节点都能访问
- 父节点中的变量**可以在子节点中使用**
- 子节点中的变量**仅在当前节点内有效**，父节点和兄弟节点都看不到

这个作用域规则是理解嵌套 CMake 的关键：把公共配置放在根节点，子节点直接引用就行了。

#### 4.1.2 添加子目录

父子节点之间的关系通过 `add_subdirectory` 命令建立：

```cmake
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```

- `source_dir`：子目录的路径，该目录下必须有 `CMakeLists.txt`
- `binary_dir`：输出文件的路径，一般不需要指定
- `EXCLUDE_FROM_ALL`：子目录下的目标不会被包含到父目录的 `ALL` 目标中，需要显式构建

### 4.2 构建多模块项目

我们的目标是将 `mathlib` 和 `strlib` 分别编译成库，再让 `test1` 和 `test2` 链接各自的库来完成功能测试。库的本质还是代码，只不过从源码形态变成了二进制形态。

#### 4.2.1 根目录

根目录的 `CMakeLists.txt` 负责两件事：**定义全局变量**和**添加子目录**。

```cmake
cmake_minimum_required(VERSION 3.0)
project(MULTI_DEMO)

# 全局变量定义
# 库文件的输出路径
set(LIB_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)
# 可执行程序的输出路径
set(EXEC_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
# 头文件目录
set(HEAD_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include)
# 库的名字
set(MATH_LIB mathlib)
set(STR_LIB strlib)
# 可执行程序的名字
set(TEST1_NAME math_test)
set(TEST2_NAME str_test)

# 添加子目录
add_subdirectory(mathlib)
add_subdirectory(strlib)
add_subdirectory(test1)
add_subdirectory(test2)
```

把路径和命名都提取成变量，子节点直接引用 `${MATH_LIB}`、`${HEAD_PATH}` 即可。好处很直接：路径变了只改根节点一处，不用挨个文件去改。

#### 4.2.2 mathlib 目录

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATHLIB)
aux_source_directory(./ SRC)
include_directories(${HEAD_PATH})
set(LIBRARY_OUTPUT_PATH ${LIB_PATH})
add_library(${MATH_LIB} STATIC ${SRC})
```

- `aux_source_directory`：收集当前目录下所有源文件
- `include_directories`：引入头文件路径，`${HEAD_PATH}` 来自根节点
- `set(LIBRARY_OUTPUT_PATH ...)`：指定库的输出目录，`${LIB_PATH}` 来自根节点
- `add_library(... STATIC ...)`：编译生成静态库

#### 4.2.3 strlib 目录

```cmake
cmake_minimum_required(VERSION 3.0)
project(STRLIB)
aux_source_directory(./ SRC)
include_directories(${HEAD_PATH})
set(LIBRARY_OUTPUT_PATH ${LIB_PATH})
add_library(${STR_LIB} SHARED ${SRC})
```

结构与 `mathlib` 几乎一样，区别在于这里用 `SHARED` 生成的是**动态库**。

> **实践建议**：选择静态库还是动态库要看实际情况。如果库体积较大、更新频繁，做成动态库更合适，多个程序可以共用同一份动态库，节省内存和磁盘空间。

#### 4.2.4 test1 目录

`test1` 链接的是 `mathlib` 静态库：

```cmake
cmake_minimum_required(VERSION 3.0)
project(MATH_TEST)
aux_source_directory(./ SRC)
include_directories(${HEAD_PATH})
link_directories(${LIB_PATH})
link_libraries(${MATH_LIB})
set(EXECUTABLE_OUTPUT_PATH ${EXEC_PATH})
add_executable(${TEST1_NAME} ${SRC})
```

这里用 `link_libraries` 链接静态库。静态库在链接阶段就被打包进可执行程序，程序启动时随之加载到内存。

#### 4.2.5 test2 目录

`test2` 链接的是 `strlib` 动态库：

```cmake
cmake_minimum_required(VERSION 3.0)
project(STR_TEST)
aux_source_directory(./ SRC)
include_directories(${HEAD_PATH})
set(EXECUTABLE_OUTPUT_PATH ${EXEC_PATH})
link_directories(${LIB_PATH})
add_executable(${TEST2_NAME} ${SRC})
target_link_libraries(${TEST2_NAME} ${STR_LIB})
```

这里用的是 `target_link_libraries`，针对单个目标精确指定链接库。动态库在链接阶段不会被打包进可执行程序，只有在运行时调用到其中的函数才会加载到内存。而且多个进程可以共用内存中的同一份动态库，所以它也叫**共享库**。

#### 4.2.6 构建项目

进入 `build` 目录，执行构建：

```shell
$ cd build
$ cmake ..
-- The C compiler identification is GNU 5.4.0
-- The CXX compiler identification is GNU 5.4.0
-- Configuring done
-- Generating done
-- Build files have been written to: .../build
```

此时 `build/` 目录下已经生成了 `mathlib/`、`strlib/`、`test1/`、`test2/` 四个子目录以及 `Makefile` 等文件。接着执行 `make`：

```shell
$ make
```

构建完成后，检查一下产物：

```shell
$ tree bin/ lib/
bin/
├── math_test
└── str_test
lib/
├── libmathlib.a
└── libstrlib.so
```

静态库 `libmathlib.a`、动态库 `libstrlib.so` 在 `lib/` 中，两个可执行程序在 `bin/` 中，与根节点里配置的路径完全一致。

> **小提示**：如果在 CMakeLists.txt 中已经指定了库的输出目录，其他模块可以直接使用。但如果需要加载外部提供的库文件，或者没有指定输出路径，就必须用 `link_directories` 把库的路径显式告诉 CMake。

---

## 5. 流程控制

CMake 的 CMakeLists.txt 支持流程控制语法，可以像写 Shell 脚本一样进行**条件判断**和**循环**。

### 5.1 条件判断

`if` 语句的基本格式：

```cmake
if(<condition>)
  <commands>
elseif(<condition>)   # 可选，可以有多个
  <commands>
else()                # 可选
  <commands>
endif()
```

`if` 和 `endif` 必须成对出现，`elseif` 和 `else` 可选。

#### 5.1.1 基本表达式

直接对常量、变量或字符串进行判断时，CMake 按以下规则取真/假：

| 判真 | 判假 |
|------|------|
| `1`、`ON`、`YES`、`TRUE`、`Y` | `0`、`OFF`、`NO`、`FALSE`、`N` |
| 非零数值 | `IGNORE`、`NOTFOUND` |
| 非空字符串 | 空字符串 |

#### 5.1.2 逻辑运算

```cmake
# 取反
if(NOT <condition>)

# 与：两个条件都为真才返回 True
if(<cond1> AND <cond2>)

# 或：至少一个条件为真就返回 True
if(<cond1> OR <cond2>)
```

#### 5.1.3 比较运算

**数值比较**：

```cmake
if(<value1> LESS <value2>)           # <
if(<value1> GREATER <value2>)        # >
if(<value1> EQUAL <value2>)          # ==
if(<value1> LESS_EQUAL <value2>)     # <=
if(<value1> GREATER_EQUAL <value2>)  # >=
```

**字符串比较**：

```cmake
if(<str1> STRLESS <str2>)            # <
if(<str1> STRGREATER <str2>)         # >
if(<str1> STREQUAL <str2>)           # ==
if(<str1> STRLESS_EQUAL <str2>)      # <=
if(<str1> STRGREATER_EQUAL <str2>)   # >=
```

#### 5.1.4 文件判断

```cmake
# 文件或目录是否存在
if(EXISTS path-to-file-or-directory)

# 是否为目录（必须是绝对路径）
if(IS_DIRECTORY path)

# 是否为软链接（相当于 Windows 的快捷方式）
if(IS_SYMLINK file-name)

# 是否为绝对路径
if(IS_ABSOLUTE path)
```

绝对路径的判断标准：Linux 下从 `/` 开始，Windows 下从盘符（如 `C:\`）开始。

#### 5.1.5 其他有用的判断

**判断元素是否在列表中**（CMake >= 3.3）：

```cmake
if(<item> IN_LIST <list_variable>)
```

**比较两个路径是否等价**（CMake >= 3.24）：

```cmake
if(<path1> PATH_EQUAL <path2>)
```

`PATH_EQUAL` 和 `STREQUAL` 都能用来比路径，但有个重要区别。手误把 `/home/user/project` 写成 `/home//user///project` 时：

```cmake
cmake_minimum_required(VERSION 3.26)
project(TEST)

if("/home//user///project" PATH_EQUAL "/home/user/project")
    message("路径相等")       # 会输出
else()
    message("路径不相等")
endif()

if("/home//user///project" STREQUAL "/home/user/project")
    message("路径相等")
else()
    message("路径不相等")     # 会输出
endif()
```

`PATH_EQUAL` 会自动剔除多余的分隔符再比较，而 `STREQUAL` 只做纯字符串比较。实际项目中建议路径比较优先用 `PATH_EQUAL`。

> `if` 命令的完整条件表达式非常丰富，详见 [CMake 官方文档](https://cmake.org/cmake/help/latest/command/if.html)。

### 5.2 循环

CMake 提供了两种循环方式：`foreach` 和 `while`。

#### 5.2.1 foreach

`foreach` 的基本语法：

```cmake
foreach(<loop_var> <items>)
    <commands>
endforeach()
```

它有四种常用的遍历方式。

**方式 1：遍历整数区间**

```cmake
foreach(<loop_var> RANGE <stop>)
```

从 0 遍历到 `stop`（**闭区间**，包含 `stop` 本身）：

```cmake
cmake_minimum_required(VERSION 3.2)
project(TEST)
foreach(i RANGE 5)
    message(STATUS "当前值: ${i}")
endforeach()
```

输出 `0 1 2 3 4 5`，共 6 次。

**方式 2：指定起止和步长**

```cmake
foreach(<loop_var> RANGE <start> <stop> [<step>])
```

遍历范围是 `[start, stop]` 闭区间，`step` 默认为 1：

```cmake
cmake_minimum_required(VERSION 3.2)
project(TEST)
foreach(i RANGE 2 10 2)
    message(STATUS "当前值: ${i}")
endforeach()
```

输出 `2 4 6 8 10`。

**方式 3：遍历列表**

```cmake
foreach(<loop_var> IN [LISTS [<lists>]] [ITEMS [<items>]])
```

`LISTS` 后面直接跟列表变量名，`ITEMS` 后面跟 `${}` 展开的值：

```cmake
cmake_minimum_required(VERSION 3.2)
project(TEST)

set(LANGS C Python Rust Go)
set(TOOLS gcc cmake make)

# 用 LISTS 遍历多个列表
foreach(item IN LISTS LANGS TOOLS)
    message(STATUS "当前值: ${item}")
endforeach()

# 用 ITEMS 遍历，需要 ${} 展开
foreach(item IN ITEMS ${LANGS} ${TOOLS})
    message(STATUS "当前值: ${item}")
endforeach()
```

两种写法输出相同。注意，如果列表中的某个元素包含空格，用双引号包起来即可：

```cmake
set(LANGS C "Objective C" Rust)
```

**方式 4：并行遍历多个列表**（CMake >= 3.17）

```cmake
foreach(<loop_var>... IN ZIP_LISTS <lists>)
```

可以把多个列表"拉链式"地对齐遍历，同时拿到每个列表的对应元素：

```cmake
cmake_minimum_required(VERSION 3.17)
project(TEST)

list(APPEND LANGS C Python Rust Go JavaScript)
list(APPEND YEARS 1972 1991 2015 2009 1995)

# 指定两个变量，分别接收两个列表的当前元素
foreach(lang year IN ZIP_LISTS LANGS YEARS)
    message(STATUS "${lang} 诞生于 ${year} 年")
endforeach()

message("=======")

# 只用一个变量时，通过 变量名_0、变量名_1 来取值
foreach(item IN ZIP_LISTS LANGS YEARS)
    message(STATUS "${item_0} 诞生于 ${item_1} 年")
endforeach()
```

几个要点：

- 变量数量应与列表数量一致，若只给一个变量，则通过 `变量名_0`、`变量名_1`……来取各列表的当前元素（下标从 0 开始）
- 如果多个列表长度不同，短的列表遍历完后不再参与后续迭代

输出：

```
-- C 诞生于 1972 年
-- Python 诞生于 1991 年
-- Rust 诞生于 2015 年
-- Go 诞生于 2009 年
-- JavaScript 诞生于 1995 年
=======
-- C 诞生于 1972 年
-- Python 诞生于 1991 年
-- Rust 诞生于 2015 年
-- Go 诞生于 2009 年
-- JavaScript 诞生于 1995 年
```

#### 5.2.2 while

`while` 循环的语法很简单，条件表达式的写法和 `if` 完全一致：

```cmake
while(<condition>)
    <commands>
endwhile()
```

一个实用的例子，逐个弹出列表元素直到清空：

```cmake
cmake_minimum_required(VERSION 3.5)
project(TEST)

set(TOOLS gcc cmake make ninja)
list(LENGTH TOOLS LEN)

while(${LEN} GREATER 0)
    message(STATUS "剩余工具: ${TOOLS}")
    list(POP_FRONT TOOLS)       # 弹出第一个元素
    list(LENGTH TOOLS LEN)      # 更新长度
endwhile()
```

输出：

```
-- 剩余工具: gcc;cmake;make;ninja
-- 剩余工具: cmake;make;ninja
-- 剩余工具: make;ninja
-- 剩余工具: ninja
```

列表长度归零后循环自然退出。