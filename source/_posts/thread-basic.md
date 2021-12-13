---
title: 线程知识点
date: 2021-10-20 16:09:55
tags:
  - java
categories:
  - interview
---
### 进程是什么
> 进程是一个具有独立功能的程序在一个数据集上的一次动态执行的过程，是操作系统调度和进行资源分配和调度的一个独立单位，进程一般由程序、数据集合、进程控制块三部分组成
+ 程序：用于描述进程要完成的功能，是控制进程执行的指令集
+ 数据集合：程序在执行时所需要的数据和工作区
+ 进程控制块：Program Control Block，简称PCB，包含进程的描述信息和控制信息，是进程存在的唯一标示
> 进程由内存空间（代码、数据、进程空间、打开的文件）和一个或者多个线程组成
### 线程是什么
> 线程是程序执行流的最小单元，是处理器调度和分派的基本单位，一个进程可以有一个或多个线程，各个线程之间共享程序的内存空间（也就是所在进程的内存空间），一个标准的线程由线程ID、当前指令指针（PC）、寄存器和堆栈组成
+ 什么是时间片
  大部分操作系统（windows、linux）的任务调度都是采用时间片轮转的方式进行抢占式调度，在一个进程中，当 一个线程任务执行几毫秒后，会有操作系统的内核进行调度，通过硬件的计数器中断处理器让该线程强制暂停并将该线程的寄存器暂存到内存中，通过查看线程列表来决定下一个线程的执行，并从内存中恢复该线程的寄存器，在这个过程中，任务执行的那一小段时间就叫时间片，任务正在执行的状态就叫运行状态（RUNNABLE），被暂停的线程任务状态就叫就绪状态（WAITING），意为等待下一个时间片的到来。
![avatar](/images/java_thread_basic/1.png)
### 进程和线程的关系
+ 线程是程序执行的最小单位，而进程是操作系统分配资源的最小单位；
+ 一个进程由一个或多个线程组成，线程是一个进程中代码的不同执行路线；
+ 进程之间相互独立，但同一进程下的各个线程之间共享程序的内存空间(包括代码段、数据集、堆等)及一些进程级的资源(如打开文件和信号)，某进程内的线程在其它进程不可见；
+ 调度和切换：线程上下文切换比进程上下文切换要快得多
![avatar](/images/java_thread_basic/2.png)
### 内核线程
> 多核处理器是指一个处理器上集成多个运算核心，从而提高计算能力，每一个处理核心对应一个内核线程
内核线程（Kernel Thread，KLT）就是直接由操作系统内核支持的线程，这种线程由内核来完成线程切换，内核通过操作调度器对线程进行调度，并负责将线程映射到各个处理器上，一般一个处理核心对应一个内核线程
### 轻量级进程
轻量级进程（Lightweight Process，LWP），轻量级进程就是我们通常意义上所讲的线程，也被叫做用户线程。由于每个轻量级进程都由一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。用户线程与内核线程的对应关系有三种模型：一对一模型、多对一模型、多对多模型，在这以4个内核线程、3个用户线程为例对三种模型进行说明。

### 线程状态介绍
> java.lang.Thread 类定义了枚举类型State，包含Thread的所有状态
```
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```
![avatar](/images/java_thread_basic/2.png)
+ 1. 初始状态(NEW)
实现Runnable接口和继承Thread可以得到一个线程类，new一个实例出来，线程就进入了初始状态。
```
// 通过Thread类
Thread thread = new Thread();
// 通过Runnable接口
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        
    }
};
```
+ 2. 就绪状态(RUNNABLE)
  + 1. READY
  就绪状态只是说你有资格运行，调度程序没有挑选到你，你就永远是就绪状态。
  调用线程的start()方法，此线程进入就绪状态。
  ```
  Thread thread = new Thread(()->{
      System.out.println("hello");
  });
  thread.start();
  ```
  当前线程sleep()方法结束，其他线程join()结束，等待用户输入完毕，某个线程拿到对象锁，这些线程也将进入就绪状态。
  当前线程时间片用完了，调用当前线程的yield()方法，当前线程进入就绪状态。
  锁池里的线程拿到对象锁后，进入就绪状态。
  + 2. RUNNING
  线程调度程序从可运行池中选择一个线程作为当前线程时线程所处的状态。这也是线程进入运行状态的唯一的一种方式。

+ 3. 阻塞状态(BLOCKED)
阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块(获取锁)时的状态。
```
public class ConcurrencyTest {
    public static void main(String[] args) {
        Object lock = new Object();
        ThreadPoolExecutor executor = new ThreadPoolExecutor(10,10,0, TimeUnit.SECONDS, new LinkedBlockingDeque<>());
        for (int i = 0; i< 11; i++) {
            Thread thread = new Thread(() -> {
                try {
                    synchronized (lock) {
                        System.out.println("try to refresh config");
                        Thread.sleep(3*60*1000);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
            executor.submit(thread);
        }
        executor.shutdown();;
    }
}
```

+ 4. 等待(WAITING)
处于这种状态的线程不会被分配CPU执行时间，它们要等待被显式地唤醒，否则会处于无限期等待的状态。
```
public static void main(String[] args) {
  Thread t = new Thread(()->{
    System.out.println("hello");
  });
  System.out.println("start");
  // 调用start方法，t线程从NEW状态-》runnable状态
  t.start();
  // 调用join方法，让main线程处于waiting状态，先执行t线程的start-》hello，t线程执行结束后处于terminated，main线程从waiting状态恢复到runnable状态，执行自己的的end
  t.join();
  System.out.println("end");
}
```
![avatar](/images/java_thread_basic/7.png)
+ 5. 超时等待(TIMED_WAITING)
处于这种状态的线程不会被分配CPU执行时间，不过无须无限期等待被其他线程显示地唤醒，在达到一定时间后它们会自动唤醒。
```
public static void main(String[] args) {
  // 当前main线程处于RUNNABLE状态
  try {
    // 调用sleep后，main线程进入TIMED_WAITING状态
      Thread.sleep(10000);
  } catch (InterruptedException e) {
      e.printStackTrace();
  }
  // 休眠结束后，恢复RUNNABLE状态
}
```
![avatar](/images/java_thread_basic/6.png)
+ 6. 终止状态(TERMINATED)
当线程的run()方法完成时，或者主线程的main()方法完成时，我们就认为它终止了。这个线程对象也许是活的，但是它已经不是一个单独执行的线程。线程一旦终止了，就不能复生。
在一个终止的线程上调用start()方法，会抛出java.lang.IllegalThreadStateException异常。

### 线程的应用实践
+ Thread类
```
public class ThreadStatusTest {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("hello");
            try {
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println("start");
        System.out.println("==1=="+t.getState());
        t.start();
        System.out.println("==2=="+t.getState());
        t.join();
        System.out.println("==3=="+t.getState());
        System.out.println("end");
    }
}
```
如上图
![avatar](/images/java_thread_basic/4.png)
![avatar](/images/java_thread_basic/5.png)