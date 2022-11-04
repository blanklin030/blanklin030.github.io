---
title: 记一次线上core dump
date: 2022-01-19 19:04:11
tags:
  - linux
  - cpp
categories:
  - sre
---
### 问题描述
官方解释
```
min_free_kbytes:
This is used to force the Linux VM to keep a minimum number
of kilobytes free.  The VM uses this number to compute a
watermark[WMARK_MIN] value for each lowmem zone in the system.
Each lowmem zone gets a number of reserved free pages based
proportionally on its size.
Some minimal amount of memory is needed to satisfy PF_MEMALLOC
allocations; if you set this to lower than 1024KB, your system will
become subtly broken, and prone to deadlock under high loads.
Setting this too high will OOM your machine instantly.
```

```
echo 3145728 >> /proc/sys/vm/min_free_kbytes
```


```
sudo sysctl -a | grep min_free
```
![docker-bridge](/images/min_free_kbytes/2.png)

```
sysctl -a | grep oom
```
![docker-bridge](/images/min_free_kbytes/1.png)


min_free_kbytes大小的影响
min_free_kbytes设的越大，watermark的线越高，同时三个线之间的buffer量也相应会增加。这意味着会较早的启动kswapd进行回收，且会回收上来较多的内存（直至watermark[high]才会停止），这会使得系统预留过多的空闲内存，从而在一定程度上降低了应用程序可使用的内存量。极端情况下设置min_free_kbytes接近内存大小时，留给应用程序的内存就会太少而可能会频繁地导致OOM的发生。

min_free_kbytes设的过小，则会导致系统预留内存过小。kswapd回收的过程中也会有少量的内存分配行为（会设上PF_MEMALLOC）标志，这个标志会允许kswapd使用预留内存；另外一种情况是被OOM选中杀死的进程在退出过程中，如果需要申请内存也可以使用预留部分。这两种情况下让他们使用预留内存可以避免系统进入deadlock状态。

