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
               if (p == head && tryAcquire(arg)) {
                  // 将这个获取成功的节点设置为头节点
                   setHead(node);
                   p.next = null; // help GC
                   failed = false;
                   return interrupted;
               }
               // 第一个语句没有进入，则走到这步，代码见 ↓ , 方法如果返回false则继续下一次循环，否则
               //通过LockSupport去park当前线程线程，并返回中断状态
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
