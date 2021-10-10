---
title: linux机器cpu打满排查过程
date: 2021-07-13 10:09:55
tags:
  - java
  - cpu
  - linux
categories:
  - devops
---

### 问题背景
接收到告警短信，web应用登陆无响应，请求api均为504
### 影响范围
平台无法正常使用
### 排除思路
一般情况下，java应用占用cpu较高的原因大部分分为以下两个情况
1. 应用属于计算密集型应用
2. 应用中出现了死循环
### 排除过程
+ 1. 首先查看系统资源占用信息，使用`top`查一下当前`cpu`较高的进程`pid`
```
top
```
![avatar](1.png)
+ 2. 查看到进程`ID`后，可以通过`ps aux | grep pid`查看进程的详情
```
ps aux | grep pid
```
![avatar](2.png)
+ 3. 定位到最耗`cpu`的进程后，使用`ps -mp pid -o THREAD,tid,time | sort -rn`命令查看线程列表，并找到占用cpu最高的线程。其中`tid`表示线程`ID`，`time`表示线程已经运行的时间
```
ps -mp pid -o THREAD,tid,time | sort -rn
```
![avatar](3.png)

+ 4. 将需要的线程`ID`转换为`16`进制格式，可以使用`printf "%x\n" tid`命令
```
printf "%x\n" tid
```
![avatar](4.png)

+ 5. 打印线程堆栈信息，可以使用命令`jstack pid | grep tid -A line`
```
jstack pid | grep tid -A line
```
![avatar](5.png)

### 附加知识
1. 什么是Futex
Futex 是Fast Userspace muTexes的缩写，按英文翻译过来就是快速用户空间互斥体。Futex是一种用户态和内核态混合的同步机制，首先，同步的进程间通过mmap共享一段内存，futex变量就位于这段共享的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的futex变量，如果没有竞争发生，则只修改futex,而不用再执行系统调用了。当通过访问futex变量告诉进程有竞争发生，则还是得执行系统调用去完成相应的处理(wait 或者 wake up)。简单的说，futex就是通过在用户态的检查，（motivation）如果了解到没有竞争就不用陷入内核了，大大提高了low-contention时候的效率。 Linux从2.5.7开始支持Futex。
```
Manual FUTEX(2)

NAME
       futex - fast user-space locking

SYNOPSIS
       #include <linux/futex.h>
       #include <sys/time.h>

       int futex(int *uaddr, int op, int val, const struct timespec *timeout,
                 int *uaddr2, int val3);
```
虽然参数有点长，其实常用的就是前面三个，后面的timeout大家都能理解，其他的也常被ignore。

    uaddr就是用户态下共享内存的地址，里面存放的是一个对齐的整型计数器。
    op存放着操作类型。定义的有5中，这里我简单的介绍一下两种，剩下的感兴趣的自己去man futex
 FUTEX_WAIT: 原子性的检查uaddr中计数器的值是否为val,如果是则让进程休眠，直到FUTEX_WAKE或者超时(time-out)。也就是把进程挂到uaddr相对应的等待队列上去。
 FUTEX_WAKE: 最多唤醒val个等待在uaddr上进程。