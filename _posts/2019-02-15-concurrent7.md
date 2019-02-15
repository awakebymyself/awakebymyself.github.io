---
layout: post
title:  "Java并发之Executor框架"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# Java并发之Executor框架

Executor框架作为一个中间者将任务的生产和执行解耦出来，简化了模型。

Executor框架成员主要有两种`ThreadPoolExecutor` 和 `ScheduledThreadPoolExecutor`， 下面将分别介绍这两种

#### ThreadPoolExecutor

我们通常由`Executors`来创建下面三种
1. `FixedThreadPool` 固定线程数, 使用的是无界队列，最大线程数参数无用,多余线程的存活时间也无效

2. `SingleThreadPool` 单线程执行任务，与有界队列类似

3. `CachedThreadPool` 根据需要创建新线程的线程池, 核心线程数为0, 存活时间1min<使用同步移交队列.
如果提交速度过快，高于任务的处理速度可能会耗尽cpu资源


#### ScheduledThreadPoolExecutor

由`Executors.newScheduledThreadPool` 创建, 内部使用`DelayedWorkQueue`.

它的实现相比较`ThreadPoolExecutor`做了部分修改。

任务在被提交的时候会被包装成`ScheduledFutureTask`
```java
private class ScheduledFutureTask<V>
          extends FutureTask<V> implements RunnableScheduledFuture<V> {

      /** Sequence number to break ties FIFO */
      private final long sequenceNumber;

      /** The time the task is enabled to execute in nanoTime units */
      private long time;


      private final long period;

      //省略
```

ScheduledFutureTask主要包含3个成员变量，如下。
*  long型成员变量time，表示这个任务将要被执行的具体时间。
* long型成员变量sequenceNumber，表示这个任务被添加到ScheduledThreadPoolExecutor中
的序号。
* long型成员变量period，表示任务执行的间隔周期。

任务队列使用`DelayedWorkQueue`

DelayQueue封装了一个PriorityQueue，这个PriorityQueue会对队列中的Scheduled-
FutureTask进行排序。排序时，time小的排在前面（时间早的任务将被先执行）。如果两个
ScheduledFutureTask的time相同，就比较sequenceNumber，sequenceNumber小的排在前面（也就
是说，如果两个任务的执行时间相同，那么先提交的任务将被先执行）。


#### `FutureTask`

 表示异步的计算过程，可以和executor结合使用(实现了runnable接口)也可以单独使用, 实现是自旋+cas修改状态。。。
