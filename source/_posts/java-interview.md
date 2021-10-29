---
title: java面试常见问题
date: 2021-10-10 10:09:55
tags:
  - java
categories:
  - interview
---

### 反射


### 泛型
+ 目的：参数化类型
+ 用法：泛型类、泛型接口、泛型方法
  + 泛型类
  ```
  // 简单范型
  class Test<T> {
    private T var;
    public T getVar() {
      return var;
    }
    public void setVar(T var) {
      this.var = var;
    }
  } 
  // 多元泛型
  class TestKV<K,V> {
    private K key;
    private V value;
    public K getKey() {
      return key;
    }
    public void setKey(K key) {
      this.key = key;
    }
    public V getValue() {
      return value;
    }
    public void setValue(V value) {
      this.value = value;
    }
  }

  public class Demo {
    public static void main(String[] args) {
      Test<String> test = new Test<String>();
      test.setVar("aa");
      System.out.println(test.getVar());

      TestKV<String, String> testKV = new TestKV<String, String>();
      testKV.setKey("hello");
      testKV.setValue("world");
      System.out.println(testKV.getKey() +":"+testKV.getValue());
    }
  }
  ```
  + 泛型接口
  ```
  interface Test<T> {
    public T getVar();
  }
  class TestImpl<T> implement Test<T> {
    private T var;
    public TEstImpl(T var) {
      this.var = var;
    }
    public T getVar() {
      return var;
    }
    public void setVar(T var) {
      this.var = var;
    }
  }
  public class Demo{
    public static void main(String[] args) {
      Test<String> test = new TestImpl<String>("tom");
      System.out.pringln(test.getVar());
    }
  }
  ```
  + 泛型方法
  ```
  public class Demo {
    public <T> T getObject(Class<T> c) {
      T t = c.newInstance();
      retrun t;
    }
  }
  ```


### 注解


### jvm调参
+ java进程启动时，未指定最大堆大小和默认初始值时，系统如何分配
```
\\ 直接启动，jvm的那些参数是如何分配的
java xx.jar
```
> 首先可以通过jinfo -flags pid查看jvm参数，可以发现
```
-XX:CICompilerCount=15 
-XX:InitialHeapSize=2147483648 
-XX:MaxHeapSize=32210157568 
-XX:MaxNewSize=10736369664 
-XX:MinHeapDeltaBytes=524288 
-XX:NewSize=715653120 
-XX:OldSize=1431830528 
-XX:+UseCompressedClassPointers 
-XX:+UseCompressedOops 
-XX:+UseFastUnorderedTimeStamps 
-XX:+UseParallelGC
```
知道答案是`-XX:InitialHeapSize=2147483648`和`-XX:MaxHeapSize=32210157568`。另外我们通过 [jvm默认配置](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size)我们发现这一段`Server JVM Default Initial and Maximum Heap Sizes
The default initial and maximum heap sizes work similarly on the server JVM as it does on the client JVM, except that the default values can go higher. On 32-bit JVMs, the default maximum heap size can be up to 1 GB if there is 4 GB or more of physical memory. On 64-bit JVMs, the default maximum heap size can be up to 32 GB if there is 128 GB or more of physical memory. You can always set a higher or lower initial and maximum heap by specifying those values directly; see the next section.`中文意思就是32位系统默认最大值可以到1GB，如果物理内存大于或者等于4GB，而在64位系统默认最大堆内存可以达到32GB或者更多，如果物理内存大袋128GB或者更多时。
但是要注意过多的

+ jvm参数设置说明
![avatar](/images/java_interview/1.png)

+ 并行收集器相关参数
![avatar](/images/java_interview/2.png)

+ JVM CMS相关参数
![avatar](/images/java_interview/3.png)

+ JVM辅助信息参数设置
![avatar](/images/java_interview/4.png)

+ JVM GC垃圾回收器参数设置
JVM给出了3种选择：串行收集器、并行收集器、并发收集器。串行收集器只适用于小数据量的情况，所以生产环境的选择主要是并行收集器和并发收集器。
默认情况下JDK5.0以前都是使用串行收集器，如果想使用其他收集器需要在启动时加入相应参数。JDK5.0以后，JVM会根据当前系统配置进行智能判断。
串行收集器
-XX:+UseSerialGC：设置串行收集器。
并行收集器（吞吐量优先）
-XX:+UseParallelGC：设置为并行收集器。此配置仅对年轻代有效。即年轻代使用并行收集，而年老代仍使用串行收集。
-XX:ParallelGCThreads=20：配置并行收集器的线程数，即：同时有多少个线程一起进行垃圾回收。此值建议配置与CPU数目相等。
-XX:+UseParallelOldGC：配置年老代垃圾收集方式为并行收集。JDK6.0开始支持对年老代并行收集。
-XX:MaxGCPauseMillis=100：设置每次年轻代垃圾回收的最长时间（单位毫秒）。如果无法满足此时间，JVM会自动调整年轻代大小，以满足此时间。
-XX:+UseAdaptiveSizePolicy：设置此选项后，并行收集器会自动调整年轻代Eden区大小和Survivor区大小的比例，以达成目标系统规定的最低响应时间或者收集频率等指标。此参数建议在使用并行收集器时，一直打开。
并发收集器（响应时间优先）
-XX:+UseConcMarkSweepGC：即CMS收集，设置年老代为并发收集。CMS收集是JDK1.4后期版本开始引入的新GC算法。它的主要适合场景是对响应时间的重要性需求大于对吞吐量的需求，能够承受垃圾回收线程和应用线程共享CPU资源，并且应用中存在比较多的长生命周期对象。CMS收集的目标是尽量减少应用的暂停时间，减少Full GC发生的几率，利用和应用程序线程并发的垃圾回收线程来标记清除年老代内存。
-XX:+UseParNewGC：设置年轻代为并发收集。可与CMS收集同时使用。JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此参数。
-XX:CMSFullGCsBeforeCompaction=0：由于并发收集器不对内存空间进行压缩和整理，所以运行一段时间并行收集以后会产生内存碎片，内存使用效率降低。此参数设置运行0次Full GC后对内存空间进行压缩和整理，即每次Full GC后立刻开始压缩和整理内存。
-XX:+UseCMSCompactAtFullCollection：打开内存空间的压缩和整理，在Full GC后执行。可能会影响性能，但可以消除内存碎片。
-XX:+CMSIncrementalMode：设置为增量收集模式。一般适用于单CPU情况。
-XX:CMSInitiatingOccupancyFraction=70：表示年老代内存空间使用到70%时就开始执行CMS收集，以确保年老代有足够的空间接纳来自年轻代的对象，避免Full GC的发生。
其它垃圾回收参数
-XX:+ScavengeBeforeFullGC：年轻代GC优于Full GC执行。
-XX:-DisableExplicitGC：不响应 System.gc() 代码。
-XX:+UseThreadPriorities：启用本地线程优先级API。即使 java.lang.Thread.setPriority() 生效，不启用则无效。
-XX:SoftRefLRUPolicyMSPerMB=0：软引用对象在最后一次被访问后能存活0毫秒（JVM默认为1000毫秒）。
-XX:TargetSurvivorRatio=90：允许90%的Survivor区被占用（JVM默认为50%）。提高对于Survivor区的使用率。

+ JVM参数疑问解答
-Xmn，-XX:NewSize/-XX:MaxNewSize，-XX:NewRatio 3组参数都可以影响年轻代的大小，混合使用的情况下，优先级是什么？
如下：
高优先级：-XX:NewSize/-XX:MaxNewSize 
中优先级：-Xmn（默认等效 -Xmn=-XX:NewSize=-XX:MaxNewSize=?） 
低优先级：-XX:NewRatio 
推荐使用-Xmn参数，原因是这个参数简洁，相当于一次设定 NewSize/MaxNewSIze，而且两者相等，适用于生产环境。-Xmn 配合 -Xms/-Xmx，即可将堆内存布局完成。
-Xmn参数是在JDK 1.4 开始支持。

+ JVM参数设置优化例子
1. 承受海量访问的动态Web应用
服务器配置：8 CPU, 8G MEM, JDK 1.6.X
参数方案：
-server -Xmx3550m -Xms3550m -Xmn1256m -Xss128k -XX:SurvivorRatio=6 -XX:MaxPermSize=256m -XX:ParallelGCThreads=8 -XX:MaxTenuringThreshold=0 -XX:+UseConcMarkSweepGC
调优说明：
-Xmx 与 -Xms 相同以避免JVM反复重新申请内存。-Xmx 的大小约等于系统内存大小的一半，即充分利用系统资源，又给予系统安全运行的空间。
-Xmn1256m 设置年轻代大小为1256MB。此值对系统性能影响较大，Sun官方推荐配置年轻代大小为整个堆的3/8。
-Xss128k 设置较小的线程栈以支持创建更多的线程，支持海量访问，并提升系统性能。
-XX:SurvivorRatio=6 设置年轻代中Eden区与Survivor区的比值。系统默认是8，根据经验设置为6，则2个Survivor区与1个Eden区的比值为2:6，一个Survivor区占整个年轻代的1/8。
-XX:ParallelGCThreads=8 配置并行收集器的线程数，即同时8个线程一起进行垃圾回收。此值一般配置为与CPU数目相等。
-XX:MaxTenuringThreshold=0 设置垃圾最大年龄（在年轻代的存活次数）。如果设置为0的话，则年轻代对象不经过Survivor区直接进入年老代。对于年老代比较多的应用，可以提高效率；如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。根据被海量访问的动态Web应用之特点，其内存要么被缓存起来以减少直接访问DB，要么被快速回收以支持高并发海量请求，因此其内存对象在年轻代存活多次意义不大，可以直接进入年老代，根据实际应用效果，在这里设置此值为0。
-XX:+UseConcMarkSweepGC 设置年老代为并发收集。CMS（ConcMarkSweepGC）收集的目标是尽量减少应用的暂停时间，减少Full GC发生的几率，利用和应用程序线程并发的垃圾回收线程来标记清除年老代内存，适用于应用中存在比较多的长生命周期对象的情况。
2. 内部集成构建服务器案例
高性能数据处理的工具应用
服务器配置：1 CPU, 4G MEM, JDK 1.6.X
参数方案：
-server -XX:PermSize=196m -XX:MaxPermSize=196m -Xmn320m -Xms768m -Xmx1024m
调优说明：
-XX:PermSize=196m -XX:MaxPermSize=196m 根据集成构建的特点，大规模的系统编译可能需要加载大量的Java类到内存中，所以预先分配好大量的持久代内存是高效和必要的。
-Xmn320m 遵循年轻代大小为整个堆的3/8原则。
-Xms768m -Xmx1024m 根据系统大致能够承受的堆内存大小设置即可。