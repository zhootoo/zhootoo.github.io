---
title: Linux网络编程
date: 2026-06-26 19:46:36
categories: [计算机网络]
tags:
  - 网络编程
  - socket
  - Linux
---
## 网络编程套接字socket
跨主机传输需要注意的问题
### 字节序问题
假设要存储的数字为：0x12345678

1- 大端存储：低地址存放高位字节，高地址存放低位字节
    0x12345678 
    地址由小到大-->
2- 小端存储：低地址存放低位字节，高地址存放高位字节
    0x78563412
    地址由小到大-->
现在的解决方法是：区分主机字节序和网络字节序
1- 主机字节序：host
2- 网络字节序：network
_to__函数：
当数据从主机字节序转换为网络字节序时，需要调用htons()函数
当数据从网络字节序转换为主机字节序时，需要调用ntohs()函数
htons 是将2字节数据转换为网络字节序
htonl 是将4字节据转换为网络字节序
ntohs 是将2字节数据转换为主机字节序
ntohl 是将4字节数据转换为主机字节序

### 对齐问题
比如：
struct
{
    int i;
    float f;
    char ch1;
    char ch2;
}
编译器默认会进行对齐，
0-3:  int i
4-7:  float f
8:    char ch
9:    char ch1
10-11: padding（2字节，凑成4的倍数），所以总和是12字节。
不同机器上对齐方式可能有区别，相同的结构体在不同机器上大小可能不一样，解析规则不太一样。所以在网络传输上，解决方法是不对齐。

### 类型长度问题
相同的类型在不同的机器上，长度可能不一样。
比如：int 在32位机器上是4字节，在16位机器上是2字节。
解决方案：int32_t, uint_32_t, int16_t, uint16_t, int8_t, uint8_t
这几个类型都是固定长度的，在不同的机器上长度是固定的。

## SOCKET是什么
介绍一下Linux 上socket的概念
int socket(int domain, int type, int protocol)
domain:
介绍几种不同的domain类型
AF_UNIX: 本地通信
AF_LOCAL:
AF_INET:Ipv4
AF_INET6:Ipv6

type:(详细介绍一下各项)
SOCK_STREAM: 流式套接字
SOCK_DGRAM: 数据报套接字
SOCK_RAW: 原始套接字
SOCK_SEQPACKET: 可靠的有序数据报套接字

## 报式套接字


## 流式套接字
