---
layout: post
title:  "Java并发之线程池"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# java并发之线程池

`背景`: 线程池应该是我们应用程序开发中使用到最多的并发技术了，比如说应用的日志，后台任务的处理都可以通过线程池异步的执行，它是我们开发过程中必不可少的工具。

`使用线程池的好处`:
 * 降低资源消耗，避免重复创建和销毁线程
 * 提高响应速度，通过异步将任务并行执行
 * 为线程的生命周期提高管理，线程池帮助我们进行分配调优(参数配置)和监控线程，避免我们手动创建。

线程池的创建通过构造函数：
```java
ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,  TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

分别解释下参数的含义：
 * `corePoolSize` 核心线程数量，创建的时候不会初始化线程除非调用`prestartAllCoreThreads`
 * `maximumPoolSize` 最大线程数, 当核心线程数不够用的时候会继续创建直到最多线程数
 * `keepAliveTime` `TimeUnit`  当线程数超过核心的数量，那么多出的线程的等待执行任务的最大空闲时间，如果任务很多并且执行时间短，可以增加时间提高线程利用率
 * `workQueue` 任务阻塞队列, 用来保存任务
 * `threadFactory` 用来配置线程池创建线程的方式
 * `RejectedExecutionHandler` 指定饱和策略, jdk提供了四种策略, `abort`直接抛出异常(默认)， `CallerRun`使用调用者线程执行, `discardOldest`抛弃队列头的任务并执行当前的，`discard`直接抛弃


 先大体说下线程池的执行流程：当任务通过`execute`或者`submit`被提交到线程池之后会经历下面几个阶段
  1. 如果当前运行的线程数小于`corePoolSize`，则创建新线程来执行任务（需要获取全局锁）
  2. 如果大于等于，则将任务加入阻塞队列`BlockingQueue`
  3. 如果阻塞队列满了，则创建新线程来执行任务（需要获取全局锁）
  4. 如果创建新线程将使当前运行的线程数超过`maximumPoolSize`，则调用拒绝策略(RejectedExecutionHandler.rejectedExecution)

  这样做的目的是避免获取全局锁，增加可伸缩性，因为大部分任务执行的时候会进入第二步。


  下面从代码层面分析下线程池是如何执行任务的， 以`execute` 为例。
```java
public void execute(Runnable command) {
       if (command == null)
           throw new NullPointerException();

       int c = ctl.get();
       // 小于核心数，创建线程并执行,线程执行完成firsttask之后会循环从任务队列获取任务执行
       if (workerCountOf(c) < corePoolSize) {
           if (addWorker(command, true))
               return;
           c = ctl.get();
       }
       if (isRunning(c) && workQueue.offer(command)) {//添加到工作队列成功
           int recheck = ctl.get();
           if (! isRunning(recheck) && remove(command))
               reject(command);
           else if (workerCountOf(recheck) == 0)
               addWorker(null, false);//增加线程数
       }
       else if (!addWorker(command, false))//工作队列饱和，新增线程失败则调用拒绝策略
           reject(command);
   }
  ```

`execute`：无法得知是否执行成功

`submit`：返回一个futuretask，可以判断是否成功，并通过`get`异步转同步


`shutdown`:通过遍历工作者线程，并中断他们(没有工作的线程)。会等待正在执行的任务完成并拒绝接受新的任务，

`shutdownNow`: 终止所有线程。如果任务不一定要执行完可以用 这个，用`isTerminated`判断是否关闭完成。`isShutdown`判断是否调用了关闭方法。


****

#### 如何合理的配置线程池 ?

1. 对于计算密集型的任务，在拥有N个处理器的系统上，当线程池的大小为N+1时，通常能实现最优的效率。(即使当计算密集型的线程偶尔由于缺失故障或者其他原因而暂停时，这个额外的线程也能确保CPU的时钟周期不会被浪费。)

2. IO密集型任务 ,
可以使用稍大的线程池，一般为2*CPU核心数。
IO密集型任务CPU使用率并不高，因此可以让CPU在等待IO的时候去处理别的任务，充分利用CPU时间。

3. 混合型任务, 可以将任务分成IO密集型和CPU密集型任务，然后分别用不同的线程池去处理。 只要分完之后两个任务的执行时间相差不大，那么就会比串行执行来的高效。 因为如果划分之后两个任务执行时间相差甚远，那么先执行完的任务就要等后执行完的任务，最终的时间仍然取决于后执行完的任务，而且还要加上任务拆分与合并的开销，得不偿失。

4. 优先级不同的可以使用`PriorityBlockingQueue`，让高优先级的先执行

5. 使用有界队列，能够提高系统稳定性，避免OOM，增强预警能力(抛异常邮箱通知等..)

****

#### 如何监控线程池 ?

1. 继承线程池，重写`beforeExecute`, `afterExecute`, `terminated`方法，可以在任务执行前后和线程池关闭的时候执行一些代码来进行监控。

2. 线程池运行是的状态, `getActiveCount`获取运行的线程的数量，`taskCount`获取总线程数
