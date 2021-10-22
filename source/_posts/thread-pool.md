---
title: 线程池相关
date: 2021-10-17 10:09:55
tags:
  - java
categories:
  - interview
---

### 如何判断程序类型
+ CPU密集型
>  `CPU` 密集型简单理解就是利用 `CPU` 计算能力的任务,比如你在内存中对大量数据进行排序

+ I/O密集型
> 但凡涉及到网络读取，文件读取这类都是 `IO` 密集型，这类任务的特点是 `CPU` 计算耗费时间相比于等待 `IO` 操作完成的时间来说很少，大部分时间都花在了等待 `IO` 操作完成上。

### 线程池设置
+ CPU 密集型任务(N+1)： 
> 这种任务消耗的主要是 `CPU` 资源，可以将线程数设置为 `N`（`CPU` 核心数）+1，比 `CPU` 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，`CPU` 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。

+ I/O 密集型任务(2N)： 
> 这种任务应用起来，系统会用大部分的时间来处理 `I/O` 交互，而线程在处理 `I/O` 的时间段内不会占用 `CPU` 来处理，这时就可以将 `CPU` 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 `2N`。

+ 参数介绍
  + corePoolSize：核心线程数，定义了最小可以同时运行的线程数量
  + maximumPoolSize：当队列中存放的任务达到队列容量时，当前可以同时运行的线程数量变为最大线程数
  + workQueue：当新任务来的时候，先判断当前运行的线程数量是否达到核心线程数（corePoolSize），如果达到，则新任务会被存放到队列中
  + keepAliveTime：当线程池中的线程数量大于核心线程池（corePoolSize）时，如果没有新任务提交，核心线程外的线程（maximumPoolSize - corePoolSize）不会被立即销毁，而是等到keepAliveTime时间后，才会被回收销毁
  + unit：keepAliveTime的时间单位（TimeUnit类中的成员变量）
  + threadFactory：executor创建线程的工厂类
  + handler：饱和策略
    > 如果当前同时运行的线程数量达到最大线程数（maximumPoolSize），并且队列（workQueue）已经满了，ThreadPoolExecutory就会执行一些策略
      + AbortPolicy：抛出 RejectExecutionExeception 来拒绝新任务加入 （默认）
      + CallerRunsPolicy：调用执行自己的线程运行拒绝任务
      + DiscardPolicy：不处理新任务，直接丢弃
      + DiscardOldestPolicy：丢弃最早的未处理的任务

![avatar](/images/java_thread_pool/1.png)

### 线程池原理分析
+ ThreadPoolExecutor
```
public class ThreadPoolExecutor extends AbstractExecutorService {
```
> ThreadPoolExecutor类 继承了 AbstractExecutorService类
```
public abstract class AbstractExecutorService implements ExecutorService {
```
> AbstractExecutorService类 实现了 ExecutorService接口
```
public interface ExecutorService extends Executor {
```
> ExecutorService接口 继承了 Executor类

+ 线程池demo
```
package blank.lin.thread;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolTest {
    private static ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            1,
            2,
            0L,
            TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<Runnable>(1)
    );

    public static void main(String[] args) {
        threadPoolExecutor.execute(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(500);
                    System.out.println("第1个任务执行完成");
                } catch (Exception exception) {
                    exception.printStackTrace();
                }
            }
        });
        printCount();
        System.out.println("加入第1个任务，线程池刚刚初始化，没有可以执行任务的核心线程，创建一个核心线程来执行任务");

        threadPoolExecutor.execute(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(500);
                    System.out.println("第2个任务执行完成");
                } catch (Exception exception) {
                    exception.printStackTrace();
                }
            }
        });
        printCount();
        System.out.println("加入第2个任务，没有可以执行任务的核心线程，且任务数大于corePoolSize，新加入任务被放在了阻塞队列中");

        threadPoolExecutor.execute(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(500);
                    System.out.println("第3个任务执行完成");
                } catch (Exception exception) {
                    exception.printStackTrace();
                }
            }
        });
        printCount();
        System.out.println("加入第3个任务，此时，阻塞队列已满，新建非核心线程执行新加入任务");

        threadPoolExecutor.execute(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(500);
                    System.out.println("第4个任务执行完成");
                } catch (Exception exception) {
                    exception.printStackTrace();
                }
            }
        });
        printCount();
        System.out.println("加入第4个任务，此时，阻塞队列已满，新建非核心线程执行新加入任务");

        try {
            Thread.sleep(600);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        printCount();
        System.out.println("第1个任务执行完毕，核心线程空闲，阻塞队列的任务被取出来，使用核心线程来执行");

        try {
            Thread.sleep(600);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        printCount();
        System.out.println("第2个任务执行完毕，核心线程空闲，非核心线程在执行第3个任务");

        try {
            Thread.sleep(600);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        printCount();
        System.out.println("第3个任务执行完毕，非核心线程被销毁，核心线程保留");

    }

    private static void printCount() {
        System.out.println("------------------------------------");
        System.out.println("当前活跃线程数:"+threadPoolExecutor.getActiveCount());
        System.out.println("当前核心线程数:"+threadPoolExecutor.getCorePoolSize());
        System.out.println("阻塞队列中的任务数:"+threadPoolExecutor.getQueue().size());
    }

}
```
+ threadPoolExecutor.execute源码分析
> 处理过程一共3步  

  + 首先判断核心线程池是否满了，未满则开启新线程执行任务，addWorker调用会原子性检查线程运行状态和线程数，并且
  > 1. If fewer than corePoolSize threads are running, try to start a new thread with the given command as its first task.  The call to addWorker atomically checks runState and workerCount, and so prevents false alarms that would add threads when it shouldn't, by returning false.
  + 如果一个任务被成功加入到队列里，也会重复检查是否应该新添加一个线程去执行，因为自上次检查后现存线程可以已死，或者线程池在进入这个方法后就关闭了，所以我们会重复检查状态，并且判断是否需要回滚队列，或者开启新线程
  > 2. If a task can be successfully queued, then we still need to double-check whether we should have added a thread (because existing ones died since last checking) or that the pool shut down since entry into this method. So we recheck state and if necessary roll back the enqueuing if stopped, or start a new thread if there are none.
  + 如果无法加入队列任务，我们会尝试新添加一个线程，如果新线程添加失败，我们知道线程被关闭了或者已经满了，所以任务会被拒绝
  > 3. If we cannot queue task, then we try to add a new thread.  If it fails, we know we are shut down or saturated and so reject the task.  
+ 源码   
```
public void execute(Runnable command) {
        // 任务为空，抛异常
        if (command == null)
            throw new NullPointerException();
         // ctl中保存的线程池当前的一些任务状态
        int c = ctl.get();
        // 上面介绍的3步从这里开始
        // 第一步，判断当前线程池中的任务数量是否小于核心线程数
        // 如果小于的话通过addWorker新建一个线程，并将任务添加到线程去执行
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 第二步，如果当前线程数已经大于等于核心线程数了
        // 通过isRunning来判断线程池状态，线程池处于RUNNING状态，才会将任务加入到workQueue队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次获取线程池状态，如果线程池不是RUNNING状态了，就需要从workQueue中移除任务
            if (! isRunning(recheck) && remove(command))
                // 执行拒绝策略
                reject(command);
            // 当前线程池为空，则重新创建线程去执行任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 第三步
        // 创建线程执行任务失败，执行相应的拒绝任务
        else if (!addWorker(command, false))
            reject(command);
    }
```
![avatar](/images/java_thread_pool/2.png)

