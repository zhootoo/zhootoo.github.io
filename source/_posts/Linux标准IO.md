---
title: 深入理解C++之Virtual实现机制
date: 2025-04-14 19:50:59
categories: [Linux]
tags: 
- Linux
---

## Linux的IO，包括input和output是一切实现的基础
程序如果需要保存数据和读取数据需要使用IO，不然很难实现一些有用的功能。
- stdio 标准IO
- sysio系统调用IO （文件IO）
该怎么选呢？如果两者在某个系统都能用，优先选择标准IO。
因为：
1-可以屏蔽掉底层kernel的差异，运行在不同的linux版本上都是ok的，移植性好
2-标准IO可以合并系统调用，有buffer和cache机制，为你的调用做加速。
标准IO函数（需要大概介绍函数功能）

stdio: FILE类型贯穿始终(介绍一下FILE类型)
fopen()：成功返回一个FILE指针，失败返回NULL，并且errno被设置,errno现在的实现已经是私有化数据，不会和其他地方发生冲突了。
（mode参数，6种方式需要介绍一下）

fclose(),fgetc(),fputc(),fgets(),fputs(),fread(),fwrite()

printf(),scanf()

fseek(),ftell(),rewind()

fflush()
