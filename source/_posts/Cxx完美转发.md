---
title: C++完美转发与引用折叠
date: 2026-10-24 15:52:44
categories: [C++]
tags:
  - C++
  - 完美转发
---

参数转发是指在C++中将函数的参数原封不动的传递给另一个函数的技术，完美转发则是在参数传递过程中避免不必要的拷贝或者移动，保持参数的原始类型，从而提升程序的性能。

如果在参数传递过程中无法正确区分左值右值，在函数内部处理时就有可能发生不必要的拷贝或者移动，会影响程序性能。

因为如果参数是左值，在目标函数中需要调用左值接口，如果是右值，需要调用移动语义相关接口，这两种不同的处理方式会有不同的开销。

尤其是转发函数，只负责转发不做实际处理，应尽量避免拷贝和移动，让目标函数直接处理最原始的参数，保留原始参数的左值和右值属性，提升程序性能。

## 问题场景

一个具体的示例：

```cpp
class Person
{
  public:
  Person(string&)
  {
    cout << "左值构造\n";
  }

  Person(string&&)
  {
    cout << "右值构造\n";
  }
};



Person make_person(string param)
{
  return Person(param);
}

string name="test";
//左值传递
make_person(name);
//右值传递
make_person(move(name));

//代码输出“左值构造 左值构造”
```

主要问题是：

1. 经过make_person的转发，Person的构造函数无法区分左值右值，看起来似乎都是左值。
2. 函数make_person会导致调用者传递的实际参数被拷贝或者移动到make_person的形参，造成了浪费。（本来就是工具人，结果又移动又拷贝，浪费了CPU资源）

## 引用折叠

### 直接改用引用的三种尝试

通过上面的例子，参数传递到make_person会发生拷贝和移动？那么如何消除这种拷贝和移动就成了一个需要解决的问题，我们会很自然想到把参数改为引用形式：

1. 左值引用
2. 右值引用
3. 常量引用

如果我们把函数改为Person make_person(string& param)，那么传递左值没问题，但是右值会报语法错误，因为左值引用无法接收右值。

如果把函数改为Person make_person(string&& param),右值引用没问题，但是左值又会出现编译错误。

如果我们把函数改为Person make_person(const string& param),那么左值右值都不会出现问题。但是return Person(param);这一行有回报错，因为常量引用为参数增加了const属性，这和Person的参数不匹配，为参数增加const属性也改变了参数的属性，将会为其增加额外的限制，我们希望的是参数原封不动传递到目标函数。

所以通过直接使用引用看来无法实现完美转发，需要另辟蹊径。我们使用C++的模板函数自动类型推导机制，让它根据实际参数类型自动推导，这个注意应该不错。

### 改用函数模板

所以把make_person函数改为模板函数：

```cpp

//如果使用T&则只能匹配左值，右值匹配不到
//如果使用T&&，发现左值右值都可以匹配到了，编译器不会报错，
//程序最后输出  “左值构造 左值构造”
template<typename T>
Person make_person(T&& param)
{
  return Person(param);
}
```

虽然最终的目标函数，也就是Person的构造还是无法区分是左值还是右值，但是至少我们解决了一个问题，就是让中间人函数的形参至少不会发生拷贝和移动。

### T&& 为什么两种值类别都能接收

可是为什么T&&既能接受左值又能接受右值呢？我们做一些实验来验证，看看现在在make_person模版函数中，T&&的真实类型：

```cpp
template<typename T>
Person make_person(T&& param)
{
  std::cout<< std::is_lvalue_reference<T>::value << std::endl;
  std::cout<< std::is_rvalue_reference<T>::value << std::endl;
  return Person(param);
}

string name="test";
//左值传递
make_person(name);
//右值传递
make_person(move(name));
//输出：1 0      0 1
```

从上面的测试可以知道T&&正确识别了传递的参数的对应属性。

可是为什么T&&为什么可以正确识别左值右值呢？

T&&是C++11引入的新语法，对 template<typename T> void f(T&& param)的函数模板，标准规定推导规则如下：

| 实参值类别 | T 推导为 | T&& 折叠后 param 类型 |
| :--- | :--- | :--- |
| 左值 | `X&` | `X&` |
| 右值 | `X` | `X&&` |

所以make_person(name)传递的实参是个左值，最终推导出的T&&类型是int&，而make_person(move(name))传递的实参是个右值，最终推导出的T&&是个int&&。

这就是C++的万能引用和引用折叠，可以当个结论记住T&&的模板参数可以最终识别出正确的左值或者右值引用。

## 完美转发

### 具名变量为什么是左值

为什么T&&可以被正确识别了，但是最终的Person构造还是不能自动调用左值右值版本呢？

因为在C++里，变量或者表达式的类型和其值类别是不同的概念，即使变量类型是右值引用，比如int&& a = getNumber(), 但是表达式a的值类别仍然是左值，因为它有一个具体的名字和身份来指向某个空间了，所以a就成了左值，虽然a的类型是右值引用int&&。

所以，虽然类型我们区分了，但是因为在这个函数里我们有一个具名对象param，所以param表达式的值类别一定是左值，这就是为什么调用Person的构造函数还是区分不开的原因。

### 用 static_cast 与 std::forward 还原类型

怎么解决呢？

一个简单的方式是对表达式再进行一次转换`(T&&)param`，因为T&&是可以正确代表左值引用和右值引用的，我们把表达式在返回时再做一次转换就行了。但是在C++中我们一般使用C++的转换`static_cast<T&&>(param)`，而不使用C语言风格。

所以最终这样就会调用最终正确的构造版本：

```cpp
template<typename T>
Person make_person(T&& param)
{
  return Person(static_cast<T&&>(param));
}
//上述输出会显示 正确的左值构造和右值构造
```

C++中提供了一个函数做这个事情，但是其实做的是跟static_cast一样的事，只不过用标准的函数会让意图更加清晰：

`std::forward<T>(param)`，它的代码内部使用的和`static_cast<T&&>`其实是一件事。
