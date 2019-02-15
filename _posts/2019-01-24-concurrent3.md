---
layout: post
title:  "Java并发之AQS"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# java并发之AQS队列同步器

基于之前的java并发编程基础和内存模型，从这篇开始我准备着重将`java.util.concurrent`包的
内容讲解一下，以备复习巩固。

首先从总体上概述一下,java并发编程分为这么几大块：
1. 锁（Lock接口
2. 同步工具 (Semphore...
3. 线程池(ForkJoinPool...
4. 原子变量(AtomicInteger...
5. Exector框架(Exectors
6. 并发容器
> 其中锁和同步工具底层实现都是依赖于AQS， 由此可见其重要性

简单来说`AQS`就是通过int类型的状态来边上同步状态的获取以及释放，内部持有FIFO的等待队列，
子类实现其protect的方法原子性的更新`state`的值，通过值判断是否获取到锁，没有则加入等待队列。

同时内部还有一个condition队列，当`await`的时候加入条件队列, `singal`的时候出列移到等待队列。

首先说下使用方法：
1. 内部定义一个静态类继承`AQS`，实现他的`protect`方法例如`tryAccquire`。
2. 同步工具的锁语义的实现依赖于同步器, 例如获取锁则调用模板方法`accquire`, 成功获取则方法返回，
失败则阻塞，此线程被构造成Node加入等待队列

下面通过源代码分析`AQS`如何实现独占锁共享锁的语义的， 先看获取独占锁的方法`accquire(int arg)`
```java
public final void acquire(int arg) {
       if (!tryAcquire(arg) &&
           acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();
   }
```

首先`tryAccquire`是抽象方法由子类重写
 1. 如果反正成功那么获取锁的方法直接返回，代表获取锁成功。
 2. 将当前线程构建成一个`Node`。 讲解下`addWaiter`
```java
Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
```
先尝试下的将新建的Node添加到队列尾部，如果没有竞争的话直接成功，否则再进行自旋`enq(node)`直到添加成功返回。

3. 在添加到等待队列之后下一步操作就是再去竞争锁资源了, 见`acquireQueued`
```java
final boolean acquireQueued(final Node node, int arg) {
       boolean failed = true;
       try {
           boolean interrupted = false;
           // 自旋
           for (;;) {
              // 拿到这个node的前置节点
               final Node p = node.predecessor();
               // 如果它的前置节点是等待队列的头节点并且accquire成功，表示获取到锁
               // 头节点表示获取同步状态成功的节点
               if (p == head && tryAcquire(arg)) {
                  // 将这个获取成功的节点设置为头节点
                   setHead(node);
                   p.next = null; // help GC
                   failed = false;
                   return interrupted;
               }
               // 第一个语句没有进入，则走到这步，代码见 ↓ , 方法如果返回false则继续下一次循环，否则
               //通过LockSupport去park当前线程线程，并检查中断状态返回
               if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   interrupted = true;
           }
       } finally {
           if (failed)
               cancelAcquire(node);
       }
   }
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
       // node节点的状态位， 默认初始为0
       int ws = pred.waitStatus;
       // 如果是signal返回true，需要park
       if (ws == Node.SIGNAL)
           /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
           return true;
       // 状态位Canceled, 需要取消
       if (ws > 0) {
           /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
           遍历node节点的前置节点知道其ws不是取消状态，并将这个节点的next引用指向自己
           do {
               node.prev = pred = pred.prev;
           } while (pred.waitStatus > 0);
           pred.next = node;
       } else {
           /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
            CAS地将pred节点状态设置为singal
           compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
       }
       return false;
   }
```

综上就是获取独占锁的内部实现了，可以看到使用的就是`CAS`和`volatile`去原子的更新节点，通过自旋去实现多次尝试，通过`park`去阻塞当前线程。


在解读了独占锁的获取源码之后，我们再看下释放的操作`release`
```java
// 子类重写抽象方法，返回true表示释放成功
if (tryRelease(arg)) {
         Node h = head;
         if (h != null && h.waitStatus != 0)
         // 唤醒后置节点,以便重新尝试获取同步状态
             unparkSuccessor(h);
         return true;
     }
     return false;
```


> 获取共享锁的实现大体上和独占锁差不多，区别就是共享锁通过tryAcquireShared>0 表示获取成功，
释放的时候子类重写tryReleaseShared的时候需要保证原子的更新状态，因为可能多个线程去释放,一般通过自旋+CAS


***AQS之响应中断锁***

实现：`acquireInterruptibly`的`doAcquireInterruptibly`方法
```java
private void doAcquireInterruptibly(int arg)
      throws InterruptedException {
      final Node node = addWaiter(Node.EXCLUSIVE);
      boolean failed = true;
      try {
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return;
              }
              // 整体和独占锁类似，在失败的时候会检查中断的状态位，如果线程被中断了则抛出异常
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  throw new InterruptedException();
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
```


***AQS之超时锁***

通过`tryAcquireNanos`, 底层调用`doAcquireNanos`， 超时锁响应中断

实现：大体和独占锁获取相同， 在超时锁获取过程中如果获取成功则和同步锁一样，失败的话则会计算
获取锁的时间的deadline,如果小于0则直接返回false, 如果大于1000ms则进行park，小于的话则进行自旋重新尝试获取。
```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
          throws InterruptedException {
      // 超时时间小于0直接false
      if (nanosTimeout <= 0L)
          return false;
      final long deadline = System.nanoTime() + nanosTimeout;
      final Node node = addWaiter(Node.EXCLUSIVE);
      boolean failed = true;
      try {
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return true;
              }
              // 超时剩余 的时间
              nanosTimeout = deadline - System.nanoTime();
              if (nanosTimeout <= 0L)
                  return false;
              if (shouldParkAfterFailedAcquire(p, node) &&
                  nanosTimeout > spinForTimeoutThreshold)
                  // 剩余的时间大于一个阈值的时候就park住，让出cpu资源，否则就进行
                  LockSupport.parkNanos(this, nanosTimeout);
              // 响应中断的处理
              if (Thread.interrupted())
                  throw new InterruptedException();
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
```  

---

****LockSupport****

从上面的分析中我们会经常用到这么一个工具类`LockSupport`，用来阻塞或者是唤醒某个线程, 它也因此成为了构建同步工具的基础。

`park`表示阻塞， `unpark`则是唤醒，同时还提供限时的阻塞`parkNanos`和`parkUntil`，以及在dump线程的时候能够提供观察的对象参数`Blocker`


---

****Condition****

使用方法:
```java
Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    lock.lock();

    try {
        condition.await();
    } catch (InterruptedException e) {

    } finally {
        lock.unlock();
    }
```

`condition`接口类似于Object的`wait`,`notify`, `notifyall`。 相同的在于使用的时候需要先获取锁，
在等待的时候会释放锁。不过condition接口还提供了不响应中断的等待。

`实现方式:`
内部同样维护了一个等待队列，每一个conditionObject就有一个队列。复用了AQS的的Node节点。

先看下`await`的实现：

```java
public final void await() throws InterruptedException {
         // 检查中断状态位
         if (Thread.interrupted())
             throw new InterruptedException();
         //将当前线程构建成一个condition节点加入等待队列
         Node node = addConditionWaiter();
         // 释放同步状态（同步队列的头节点），也就是释放锁,唤醒同步队列的后继节点
         int savedState = fullyRelease(node);
         int interruptMode = 0;
         // signal之后会在同步队列中，则退出while循环
         while (!isOnSyncQueue(node)) {
             LockSupport.park(this);
             if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                 break;
         }
         // 竞争同步状态
         if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
             interruptMode = REINTERRUPT;
         if (node.nextWaiter != null) // clean up if cancelled
             unlinkCancelledWaiters();
         if (interruptMode != 0)
             reportInterruptAfterWait(interruptMode);
     }
```

`signal`:

调用`condition.signal()`将会唤醒在等待队列等待时间最长的节点（头节点），在唤醒前会将节点移动到同步队列中。
```java
public final void signal() {
          //必须要先获取锁才能唤醒
          if (!isHeldExclusively())
              throw new IllegalMonitorStateException();
          // 唤醒头节点,移动到同步队列中，并使用locksupport唤醒，并重新加入到同步状态的竞争中，直到成功获取同步状态,await方法才会返回。
          Node first = firstWaiter;
          if (first != null)
              doSignal(first);
      }
```      

`signalAll`:相当于等待队列中对每个节点使用`singal`，效果就是将每个节点加入到同步队列中并唤醒。
