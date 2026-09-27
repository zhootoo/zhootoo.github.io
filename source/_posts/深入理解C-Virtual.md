---
title: 深入理解C++之Virtual实现机制
date: 2026-07-14 14:50:59
categories: [C++]
tags: 
- C++
- 编程语言
---

## 虚函数引入后Class的变化

C++中一个Class一旦引入了一个Virtual 函数会发生什么变化呢？下面做一个实验。

```cpp
class A{
};
A a;
cout<<sizeof(a)<<endl; //打印值为1，一个class对象即使是空类只要占据存储空间，size就不会为0，不然根本无法获得地址。
```

加上两个普通的成员函数后的情况
```cpp
class A{
     public:
     void func1(){}
     void func2(){}
};
A a;
cout<<sizeof(a)<<endl; //打印值仍然为1，说明普通成员函数并不占据对象内存空间
```

加上一个virtual 函数
```cpp
class A{
     public:
     void func1(){}
     void func2(){}
     virtual void func3(){}
};
A a;
cout<<sizeof(a)<<endl; //打印值为8，一个64位机器，一个指针的大小为8
```

当一个类中出现了virtual 函数后，编译器会给类加一个看不见的成员变量 `void *vptr`虚函数表指针（vptr）,这是个指针，占用对象的存储空间。

## 虚函数表

当class A中至少有一个虚函数的时候，编译器就会为class A生成一个虚函数表（virtual table,vtbl），这个虚函数表会一直伴随着类A,在经过编译链接后，直到生成一个可执行文件后，类A以及伴随类A的虚函数表都会保存到可执行文件中。当可执行文件执行的时候，类A的虚函数表会被加载到内存中。

## 虚函数表指针vptr

vptr何时赋值呢？在编译期间，编译器会为在class A的构造函数中安插对vptr进行赋值的语句：
```cpp
A::A(){
    vptr = &A::vtbl;
}
```

## 类对象在内存中的布局

假设class A最终的定义如下：
```cpp
class A{
public:
     void func1(){}
     virtual void vfunc(){}
     virtual void vfunc2(){}
     virtual ~A(){}
private:
     int a;
     int b;
};
```
class A的实例在内存中的布局如下：
{% asset_img a-instance-memory.png class A 实例的内存布局 %}

## 多态性

### 表现形式
1. 程序中既存在父类也存在子类，父类中必须含有虚函数，子类中也必须重写父类中已有的虚函数。
2. 父类指针指向子类对象，或者父类引用指向子类对象。
3. 当通过父类的指针或者引用调用子类中重写的虚函数时，调用的是子类中重写的虚函数，就发生了多态。

### 多态的内存布局

```cpp
//父类
class Base{
     virtual f(){}
     virtual g(){}
     virtual h(){}
     virtual ~Base(){}
};
//子类
class Derived:public Base{
     virtual f(){}
};
```

上面的例子中只有函数f被重写了，所以Derived class实例的内存布局如下,vptr内存永远在对象的最开始位置：
{% asset_img deriveclass.png Derived class 实例的内存布局 %}