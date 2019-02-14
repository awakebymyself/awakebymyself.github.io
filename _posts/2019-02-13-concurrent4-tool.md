---
layout: post
title:  "Java并发之同步工具"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# java并发之同步工具

* 等待多线程完成的`countdownLatch`

  它允许一个线程多个等待其他线程完成操作，在`await`上等待直到调用`countdown`

  它接受一个int类型的参数，每当countdown的时候-1，直接为0才从await返回，这个参数不可以重置，如果需要使用重置的推荐使用栅栏。

  使用场景：比如说应用程序启动时候多线程检查运行必备的环境，都检查完成之后主启动类恢复执行。

  `实现方式`：

  内部也是通过AQS代理实现其语义.
```java
/**
    * Synchronization control For CountDownLatch.
    * Uses AQS state to represent count.
    */
   private static final class Sync extends AbstractQueuedSynchronizer {
       private static final long serialVersionUID = 4982264981922014374L;

       Sync(int count) {
           setState(count);
       }

       int getCount() {
           return getState();
       }
       // 实现了共享的获取和释放的方法
       // await的时候会判断状态值是否为0,0则成功并返回，否则失败加入同步队列
       protected int tryAcquireShared(int acquires) {
           return (getState() == 0) ? 1 : -1;
       }
       // 因为存在多个线程一起释放的可能，所以自旋设置状态，每次countdown则-1, 状态为0则成功
       // 成功的时候aqs会唤醒所有在共享状态等待的线程。
       protected boolean tryReleaseShared(int releases) {
           // Decrement count; signal when transition to zero
           for (;;) {
               int c = getState();
               if (c == 0)
                   return false;
               int nextc = c-1;
               if (compareAndSetState(c, nextc))
                   return nextc == 0;
           }
       }
   }
```

* 同步屏障栅栏 `CyclicBarrier`

  描述：是一个可循环使用的barrier, 作用是让一组线程到达栅栏(同步点)的时候阻塞，直到最后一个线程到达的时候，栅栏打开让所有的线程继续运行。

  使用方法:
  `CyclicBarrier cyclicBarrier = new CyclicBarrier(3);`

  构造函数接收一个int型参数表示栅栏要拦截的线程数，每个线程调用`await`方法告诉栅栏我已经到达，并被阻塞。

  使用场景: 比如多线程计算数据，最后合并计算结果

  `实现方式`：在调用`await`的时候会先获取独占锁
  ```java
  throws InterruptedException, BrokenBarrierException,
          TimeoutException {
   final ReentrantLock lock = this.lock;
   // 获取独占锁
   lock.lock();
   try {
       final Generation g = generation;
       // broken则跑异常
       if (g.broken)
           throw new BrokenBarrierException();
      // 检查中断位
       if (Thread.interrupted()) {
           breakBarrier();
           throw new InterruptedException();
       }
       // 每个线程调用就将计数-1
       int index = --count;
       // 0表示需要唤醒所有等待的线程
       if (index == 0) {  // tripped
           boolean ranAction = false;
           try {
               final Runnable command = barrierCommand;
               if (command != null)
                   command.run();
               ranAction = true;
               nextGeneration();
               return 0;
           } finally {
               if (!ranAction)
                   breakBarrier();
           }
       }
      // 不是0则表示有线程未到达
       // loop until tripped, broken, interrupted, or timed out
       for (;;) {
           try {
              // 如果不是限时的则直接等待
               if (!timed)
                   trip.await();
               else if (nanos > 0L)
                    // 否则限时的等待
                   nanos = trip.awaitNanos(nanos);
           } catch (InterruptedException ie) {
               if (g == generation && ! g.broken) {
                   breakBarrier();
                   throw ie;
               } else {
                   // We're about to finish waiting even if we had not
                   // been interrupted, so this interrupt is deemed to
                   // "belong" to subsequent execution.
                   Thread.currentThread().interrupt();
               }
           }

           if (g.broken)
               throw new BrokenBarrierException();

           if (g != generation)
               return index;

           if (timed && nanos <= 0L) {
              // 超时的话就破坏栅栏并唤醒所有等待的线程
               breakBarrier();
               throw new TimeoutException();
           }
       }
   } finally {
       lock.unlock();
   }
```

  **闭锁和栅栏的区别**
 1. countdown的计数器只能使用一次，栅栏的计数器可以reset，这样实现重用
 2. 栅栏还提供其他有用的方法比如说获取被阻塞的线程数，isBroken可以判断线程是否被中断。


* 控制线程并发数的信号量Semaphore

  使用方法:构造函数接收int类型的许可证数量，每当线程使用的时候调用`accquire`获取许可证，用完之后调用`release`释放许可。当只有一个许可的时候可以当独占锁使用


  使用场景：控制资源池使用的最大线程数，比如启动N个线程获取数据但是数据库最多十个连接，那么只能控制最多十个线程可以获取到数据库连接。


  实现原理：也是基于AQS实现

  以非公平获取许可为例, 先看下acquire的实现
  ```java
  final int nonfairTryAcquireShared(int acquires) {
           for (;;) {
               // 获取可用许可数量
               int available = getState();
               // 计算剩余的
               int remaining = available - acquires;
               // 如果许可证不足则为负数，失败并加入同步队列等待
               if (remaining < 0 ||
                   compareAndSetState(available, remaining))
                   //否则更新状态，自旋是因为可能存在多线程accquire
                   return remaining;
           }
       }
  ```

下面看下释放的规则:
```java
protected final boolean tryReleaseShared(int releases) {
           for (;;) {
               int current = getState();
               int next = current + releases;
               if (next < current) // overflow
                   throw new Error("Maximum permit count exceeded");
               // 自旋更新状态,释放成功唤醒阻塞的线程
               if (compareAndSetState(current, next))
                   return true;
           }
       }
```       

* 线程间交换数据的Exchanger
  使用方式：用于在两个线程间交换数据，通过`exchange`. 一个线程调用会阻塞直到有另外一个线程来交换数据。

  `demo`:
```java
Exchanger<String> exchanger = new Exchanger<>();
           ExecutorService executorService = Executors.newFixedThreadPool(2);

           executorService.execute(() -> {
               String a = "流水A";
               try {
                   String exchange = exchanger.exchange(a);
                   System.out.println(exchange);//银行流水B
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           });

           executorService.execute(() -> {
               String b = "银行流水B";
               String a = null;
               try {
                   a = exchanger.exchange(b);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
               System.out.println(a);//流水A
           });
```

  实现方式：see [实现原理](https://www.jianshu.com/p/c523826b2c94)
