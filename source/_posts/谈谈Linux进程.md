---
title: 谈谈Linux进程
date: 2026-08-14 13:10:12
categories: [Linux]
tags: 
- Linux
---

## 孤儿进程
当父进程提前终止，而子进程仍在运行，子进程失去父进程的监管，成为孤儿进程，系统将孤儿进程的父进程设置为init进程，init进程会收养孤儿进程，负责在其结束后回收资源。因为在Linux操作系统中子进程的资源都是通过父进程来监管的，所以孤儿进程会被init进程收养。

```C++
//孤儿进程示例
#include <unistd.h>
int main()
{
    if(fork()==0)
    {
        sleep(10);
        printf("parent process id: %d\n",getppid());

    }
    else
    {
        exit(0);
    }
}

//程序输出：parent process id: 1
```
## 僵尸进程
子进程终止了但是父进程没有通过wait或者waitpid来回收子进程的资源，子进程的进程PCB仍然保存在系统中，占用着进程表，这种进程称之为僵尸进程。僵尸进程太多可能导致无法在继续创建新的进程,PID耗尽，资源泄露，而且会拖慢系统。
解放方案：
- 主动调用wait或者waitpid来回收子进程的资源
- 终止父进程：kill -9 父进程pid，使得僵尸进程回收
- 信号处理：父进程注册SIGCHILD信号处理函数，在信号处理函数中调用wait或者waitpid来回收子进程的资源