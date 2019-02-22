---
layout: post
title:  "Java并发之并发容器"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# java并发之并发容器


### ConcurrentHashMap

这是一个线程安全的容器，关于它的原理我在网上找到一份很详细的文章，就直接引用过来了。
[分析案例](http://www.importnew.com/28263.html)

---

### ConcurrentLinkedQueue 线程安全的队列
实现也是CAS，比如在`offer`的时候通过自旋+CAS更新尾节点，如果失败重试再更新。
[ConcurrentLinkedQueue](http://www.importnew.com/27052.html)

还有`ConcurrentSkipListMap` 线程安全的跳表以及`ConcurrentSkipListSet` 。

另外还有`CopyOnWriteArrayList`用了做线程安全的`ArrayList`，适用于多读的场景，写会重新拷贝。
, `CopyOnWriteArraySet`。 问题是存在内存占用可用引发高频率GC，并且只能保证数据最终一致性。

---

### 阻塞队列BlockingQueue
##### 什么是阻塞队列？
当队列为空的时候获取会被阻塞，当队列满的时候添加也会被阻塞, 最常用于生产-消费模型

Java提供了一共七种阻塞队列
* ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。
* LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。
* PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。
* DelayQueue：一个使用优先级队列实现的无界阻塞队列。
* SynchronousQueue：一个不存储元素的阻塞队列。
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。


实现原理：lock+condition ， 运用等待通知模式实现,代码就不贴了还是比较易懂的。

---

### ForkJoin框架
#### 是什么？
Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干
个小任务，最终汇总每个小任务结果后得到大任务结果的框架。
我们再通过Fork和Join这两个单词来理解一下Fork/Join框架。Fork就是把一个大任务切分
为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结
果。

`工作窃取算法`
工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。那么，为什么
需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，可以把这个任务分割为若干
互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个
队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应。比如A线程负责处理A
队列里的任务。但是，有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有
任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列
里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被
窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿
任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

优点：充分利用多线程并行计算减少竞争
缺点：当双端队列只有一个任务的时候还是会竞争，并且消耗更多资源，因为创建了多个线程和双端队列.

详细的内容我就不一一介绍了，直接引用
[框架解读](https://www.ibm.com/developerworks/cn/java/j-jtp11137.html)
