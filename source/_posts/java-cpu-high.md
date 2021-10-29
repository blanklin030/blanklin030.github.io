---
title: java应用cpu打满排查过程
date: 2021-07-13 10:09:55
tags:
  - java
  - cpu
  - linux
categories:
  - devops
---
### 查询cpu的过程
+ 1. 查看应用的pid,我的应用名称是以dlap开头
jps的命令参数如下，`jps [options] [hostid]` 
       + -q 不输出类名、Jar名和传入main方法的参数
       + -m 输出传入main方法的参数
       + -l 输出main类或Jar的全限名
       + -v 输出传入JVM的参数
```
jps -m -l
或者
ps -ef | grep dlap | grep -v grep
或者
ps aux | grep dlap | grep -v greppid
```
![avatar](/images/java_high_cpu/1.png)

+ 2. 查看系统资源占用信息，使用`top`查一下当前进程`pid`占用较高的`cpu`线程
```
top -Hp pid
// 也可以用以下两个命令
ps -Lfp pid
// 或者
ps -mp pid -o THREAD, tid, time | sort -rn
```
![avatar](/images/java_high_cpu/2.png)

+ 3. 将需要的线程`ID`转换为`16`进制格式，可以使用`printf "%x\n" tid`命令
```
printf "%x\n" tid
```
![avatar](/images/java_high_cpu/3.png)

+ 4. 打印线程堆栈信息，可以使用命令`jstack pid | grep tid`，注意这里的tid是线程ID的16进制值
```
jstack pid | grep tid
```
![avatar](/images/java_high_cpu/4.png)

### jmap命令介绍
> jmap可以到处堆内容，然后使用jhat进行分析，语法如下  
```
NAME
       jmap - Prints shared object memory maps or heap memory details for a process, core file, or remote debug server. This command is
       experimental and unsupported.

SYNOPSIS
       jmap [ options ] pid
       jmap [ options ] executable core
       jmap [ options ] [ pid ] server-id@ ] remote-hostname-or-IP
```
+ 根据pid查看堆内存使用情况，包括使用的GC算法、堆配置参数和各代中堆内存使用情况
```
jmap -heap pid
```
![avatar](/images/java_high_cpu/5.png)

+ 根据pid查看堆内存中的对象数目、大小统计直方图，如果带上live则只统计活对象
```
jmap -histo:live pid | more
```
![avatar](/images/java_high_cpu/6.png)
> class name是对象类型，说明如下
```
B  byte
C  char
D  double
F  float
I  int
J  long
Z  boolean
[  数组，如[I表示int[]
[L+类名 其他对象
```
+ 用jmap把进程内存使用情况dump到文件中，再用jhat分析查看
```
jmap -dump:format=b,file=dumpFileName pid
```
![avatar](/images/java_high_cpu/7.png)
+ 用jhat查看
```
jhat -port 9998 /tmp/dump.dat
// 注意如果Dump文件太大，可能需要加上-J-Xmx512m这种参数指定最大堆内存
jhat -J-Xmx512m -port 9998 /tmp/dump.dat
```
![avatar](/images/java_high_cpu/8.png)
> 然后用浏览器打开，http://ip:9998

### jstat(JVM统计监测工具)
```
NAME
       jstat - Monitors Java Virtual Machine (JVM) statistics. This command is experimental and unsupported.

SYNOPSIS
           jstat [ generalOption | outputOptions vmid [ interval[s|ms] [ count ] ]

       generalOption
           A single general command-line option -help or -options. See General Options.

       outputOptions
           One or more output options that consist of a single statOption, plus any of the -t, -h, and -J options. See Output Options.

       vmid
           Virtual machine identifier, which is a string that indicates the target JVM. The general syntax is the following:

               [protocol:][//]lvmid[@hostname[:port]/servername]

           The syntax of the vmid string corresponds to the syntax of a URI. The vmid string can vary from a simple integer that
           represents a local JVM to a more complex construction that specifies a communications protocol, port number, and other
           implementation-specific values. See Virtual Machine Identifier.

       interval [s|ms]
           Sampling interval in the specified units, seconds (s) or milliseconds (ms). Default units are milliseconds. Must be a positive
           integer. When specified, the jstat command produces its output at each interval.

       count
           Number of samples to display. The default value is infinity which causes the jstat command to display statistics until the
           target JVM terminates or the jstat command is terminated. This value must be a positive integer.
```
+ 根据pid 间隔250ms 采样条数4 输出GC信息
```
jstat -gc pid 250 4
```
![avatar](/images/java_high_cpu/9.png)
> jvm 堆内容布局
![avatar](/images/java_high_cpu/10.png)
```
堆内存 = 年轻代 + 年老代 + 永久代  
年轻代 = Eden区 + 两个Survivor区（From和To）
```
> 各列含义
```
S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
EC、EU：Eden区容量和使用量
OC、OU：年老代容量和使用量
PC、PU：永久代容量和使用量
YGC、YGT：年轻代GC次数和GC耗时
FGC、FGCT：Full GC次数和Full GC耗时
GCT：GC总耗时
```

### hprof用法
> hprof 能展现cpu使用率，堆内存使用情况，语法格式如下
```
java -agentlib:hprof[=options] ToBeProfiledClass
java -Xrunprof[:options] ToBeProfiledClass
javac -J-agentlib:hprof[=options] ToBeProfiledClass
```
> 完整的命令格式
```
Option Name and Value  Description                    Default
---------------------  -----------                    -------
heap=dump|sites|all    heap profiling                 all
cpu=samples|times|old  CPU usage                      off
monitor=y|n            monitor contention             n
format=a|b             text(txt) or binary output     a
file=<file>            write data to file             java.hprof[.txt]
net=<host>:<port>      send data over a socket        off
depth=<size>           stack trace depth              4
interval=<ms>          sample interval in ms          10
cutoff=<value>         output cutoff point            0.0001
lineno=y|n             line number in traces?         y
thread=y|n             thread in traces?              n
doe=y|n                dump on exit?                  y
msa=y|n                Solaris micro state accounting n
force=y|n              force output to <file>         y
verbose=y|n            print messages about dumps     y
```
+ 每隔20毫秒采样CPU消耗信息，堆栈深度为3
```
java -agentlib:hprof=cpu=samples,interval=20,depth=3 Hello.java
```
+ 获取CPU消耗信息，能够细到每个方法调用的开始和结束，它的实现使用了字节码注入技术（BCI）
```
javac -J-agentlib:hprof=cpu=times Hello.java
```

```
javac -J-agentlib:hprof=heap=sites Hello.java
```

```
javac -J-agentlib:hprof=heap=dump Hello.java
```
> 注意在JVM启动参数中加入-Xrunprof:heap=sites参数可以生成CPU/Heap Profile文件，但对JVM性能影响非常大，不建议在线上服务器环境使用

### jinfo查看jvm启动参数
```
// 看出所有参数
jinfo -flags pid
// 查看某个具体参数,例如InitialHeapSize
jinfo -flag InitialHeapSize pid
```
![avatar](/images/java_high_cpu/11.png)
+ 开启/关闭某个jvm参数
> 使用jinnfo可以在不重启虚拟机的情况下，动态修改jvm的参数，这个方法在生产环境尤其特别有用
```
// jinfo -flag [+|-]name pid
jinfo -flag +PrintGC pid
jinfo -flag -PrintGC pid
```
+ 修改某个JVM进程的值
```
// jinfo -flag name=value pid
jinfo -flag InitialHeapSize=64g pid
```
> 注意并不是所有参数都支持动态修改
+ 查看当前jvm进程所有的系统属性
```
jinfo -sysprops pid
```
![avatar](/images/java_high_cpu/12.png)

### free命令查看机器物理内存

```
OPTIONS
       -b, --bytes
              Display the amount of memory in bytes.

       -k, --kilo
              Display the amount of memory in kilobytes.  This is the default.

       -m, --mega
              Display the amount of memory in megabytes.

       -g, --giga
              Display the amount of memory in gigabytes.

       --tera Display the amount of memory in terabytes.

       -h, --human
              Show all output fields automatically scaled to shortest three digit unit and display the units of print out.  Following units are used.

                B = bytes
                K = kilos
                M = megas
                G = gigas
                T = teras

              If unit is missing, and you have petabyte of RAM or swap, the number is in terabytes and columns might not be aligned with header.

       -w, --wide
              Switch to the wide mode. The wide mode produces lines longer than 80 characters. In this mode buffers and cache are reported in two separate columns.

       -c, --count count
              Display the result count times.  Requires the -s option.

       -l, --lohi
              Show detailed low and high memory statistics.

       -s, --seconds seconds
              Continuously display the result delay seconds apart.  You may actually specify any floating point number for delay, usleep(3) is used for microsecond  resolu‐
              tion delay times.

       --si   Use power of 1000 not 1024.

       -t, --total
              Display a line showing the column totals.

       --help Print help.

       -V, --version
              Display version information.

```
![avatar](/images/java_high_cpu/13.png)

