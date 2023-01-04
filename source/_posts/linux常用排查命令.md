---
title: 常用的linux排查命令
date: 2023-01-04 23:04:11
tags:
  - sre
categories:
  - tool
---
## 如何看查占用cpu最多的进程？

+ 方法一
核心指令：ps
实际命令：
```
ps H -eo pid,pcpu | sort -nk2 | tail
```
执行效果如下：
```
[work@test01 ~]$ ps H -eo pid,pcpu | sort -nk2 | tail
31396  0.6
31396  0.6
31396  0.6
31396  0.6
31396  0.6
31396  0.6
31396  0.6
31396  0.6
30904  1.0
30914  1.0
```
结果：
瞧见了吧，最耗cpu的pid=30914。
画外音：实际上是31396。

+ 方法二
核心指令：top
实际命令：
```
top
Shift + t
```
## 找到了最耗CPU的进程ID，对应的服务名是什么呢？
+ 方法一
核心指令：ps
实际命令：
```
ps aux | fgrep pid
```
执行效果如下：
```
[work@test01 ~]$ ps aux | fgrep 30914
work 30914  1.0  0.8 309568 71668 ?  Sl   Feb02 124:44 ./router2 –conf=rs.conf
```
结果：
瞧见了吧，进程是./router2

+ 方法二
直接查proc即可。
实际命令：
```
ll /proc/pid
```
执行效果如下：
```
[work@test01 ~]$ ll /proc/30914
lrwxrwxrwx  1 work work 0 Feb 10 13:27 cwd -> /home/work/im-env/router2
lrwxrwxrwx  1 work work 0 Feb 10 13:27 exe -> /home/work/im-env/router2/router2
```
画外音：这个好，全路径都出来了。

## 如何查看某个端口的连接情况？

+ 方法一
核心指令：netstat
实际命令：
```
netstat -lap | fgrep port
```
执行效果如下：
```
[work@test01 ~]$ netstat -lap | fgrep 22022
tcp        0      0 1.2.3.4:22022          *:*                         LISTEN      31396/imui
tcp        0      0 1.2.3.4:22022          1.2.3.4:46642          ESTABLISHED 31396/imui
tcp        0      0 1.2.3.4:22022          1.2.3.4:46640          ESTABLISHED 31396/imui
```

+ 方法二
核心指令：lsof
实际命令：
```
lsof -i :port
```
执行效果如下：
```
[work@test01 ~]$ /usr/sbin/lsof -i :22022
COMMAND   PID USER   FD   TYPE   DEVICE SIZE NODE NAME
router  30904 work   50u  IPv4 69065770       TCP 1.2.3.4:46638->1.2.3.4:22022 (ESTABLISHED)
router  30904 work   51u  IPv4 69065772       TCP 1.2.3.4:46639->1.2.3.4:22022 (ESTABLISHED)
router  30904 work   52u  IPv4 69065774       TCP 1.2.3.4:46640->1.2.3.4:22022 (ESTABLISHED)
```
学废了吗？