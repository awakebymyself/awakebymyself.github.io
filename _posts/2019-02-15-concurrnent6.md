---
layout: post
title:  "Java并发之原子变量"
categories: concurrent
tags:  concurrent
author: Lzg
---

* content
{:toc}

---

# java并发之原子变量

`背景`:在多线程更新一个类变量比如说`int a = 1`的时候， 如果两个线程同时+1可能会出现结果为2的情况，在jdk 1.5之前需要使用锁保证线程安全，但是现在可以使用原子操作类，相比锁更加高效简洁。

`原子变量更新基本类型` 提供了三个操作类
 1. `AtomicBoolean`
 2. `AtomicInteger`
 3. `AtomicLong`

这三种提供的方法几乎一致，以`AtomicInteger`为例，原子更新操作`getAndIncrement`,先获取再增加

看下实现:
```java
public final int getAndIncrement() {
       return unsafe.getAndAddInt(this, valueOffset, 1);
   }
```

每个原子类都有一个`volatile`的状态值`value`，底层通过`UnSafe`原子更新
```java
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        //自旋
        do {
            var5 = this.getIntVolatile(var1, var2); //获取当前的volatile value值
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));//cas的方式加，失败说明值被其他线程更新了，重新尝试

        return var5;
    }
```


`原子更新数组`
1. `AtomicIntegerArray` 原子更新int数组元素
1. `AtomicLongArray`
1. `AtomicReferenceArray`

以`AtomicIntegerArray`为例, demo：
```java
int a[] = new int[]{1, 2};
        AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(a);
        System.out.println(atomicIntegerArray.addAndGet(0,1));//2
        System.out.println(a[0]);//1 底层实现将数组copy了所以不变
```

`原子更新引用类型`

1. `AtomicReference` 原子更新引用类型
2. `AtomicReferenceFieldUpdater` 原子更新引用类型字段
3. `AtomicMarkableReference` 原子更新带有标记位的引用类型，可以原子更新一个布尔类型的标记位和引用类型

原理类型...

`原子更新字段类`:
1. `AtomicIntegerFieldUpdater` 原子的更新某个类的某个字段
2. `AtomicLongFieldUpdater`、
3. `AtomicStampedReference`原子更新带有版本号的引用类型,可以解决ABA问题


demo:
```java
AtomicIntegerFieldUpdater<User> fieldUpdater = AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
      User user = new User("conan", 10);

      int andIncrement = fieldUpdater.getAndIncrement(user);
      System.out.println(andIncrement);//10

      System.out.println(user.getAge());//11


  }

  private static class User {
      private final String name;
      public volatile int age; // must int volatile public

      public User(String name, Integer age) {
          this.name = name;
          this.age = age;
      }

      public Integer getAge() {
          return age;
      }
  }
```
