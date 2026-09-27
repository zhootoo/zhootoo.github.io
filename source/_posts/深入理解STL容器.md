---
title: 深入理解STL容器
date: 2026-09-26 20:59:17
categories: [C++]
tags: 
- C++
- 编程语言
---

## STL为啥出现

在很久以前，软件开发人员从事软件开发只能重复造轮子，因为最基本的数据结构和算法从来都没有一套标准。为了建立数据结构和算法的一套标准，并且降低他们之间的耦合关系（说人话就是，算法要通用，不要强绑定到一种数据结构上。比方说排序，对所有结构其实都是一样的，没必要每次搞一个数据结构都要重新开发），提升各自的独立性，弹性，STL诞生了。

## STL的六大组件
1. 容器：存放数据
2. 算法：操作数据
3. 迭代器：算法只能借助于迭代器来操作容器内部的数据
4. 仿函数：为算法提供更多的策略
5. 适配器：为算法提供更多参数的接口
6. 空间适配器：为算法和容器动态分配管理空间

## string容器常见操作

### 1. string 构造函数

```cpp
string();                 //创建一个空的字符串 例如：string str;
string(const string& str);//使用一个 string 对象初始化另一个 string 对象
string(const char* s);    //使用字符串 s 初始化
string(int n, char c);    //使用 n 个字符 c 初始化
```

### 2. string 基本赋值操作

```cpp
string& operator=(const char* s);       //char* 类型字符串 赋值给当前的字符串
string& operator=(const string& s);     //把字符串 s 赋给当前的字符串
string& operator=(char c);              //字符赋值给当前的字符串
string& assign(const char* s);          //把字符串 s 赋给当前的字符串
string& assign(const char* s, int n);   //把字符串 s 的前 n 个字符赋给当前的字符串
string& assign(const string& s);        //把字符串 s 赋给当前的字符串
string& assign(int n, char c);          //用 n 个字符 c 赋给当前字符串
string& assign(const string& s, int start, int n);//将 s 从 start 开始 n 个字符赋值给字符串
```

### 3. string 存取字符操作

```cpp
char& operator[](int n);//通过 [] 方式取字符
char& at(int n);        //通过 at 方法获取字符
```
