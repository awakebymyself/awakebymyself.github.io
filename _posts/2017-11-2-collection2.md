---
layout: post
title:  "Java集合框架源码分析(二) LinkedList"
categories: Collection
tags:  集合
author: Lzg
---

`OK, 废话不多说, 今天要说的就是LinkedList`

LinkedList实现了List接口,同时也现实了Deque接口,既是表也是双端队列, 里面的实现采用的是双向
链表, 每当添加一个元素的时候, 都需要new一个节点对象,比ArrayList更占用空间.

先看一下节点对象Node的 组成吧

```Java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
每个节点对象保存着前一个节点对象的引用和后面一个的, 所以我们在其中添加.删除的时候只需要修改节点对象
里面的引用就可以了.

LinkedList对象初始化的时候有两个节点分别对应头节点和尾节点`transient Node<E> last`
`transient Node<E> first`

先来看一下add(E) 操作
  ```java
  public boolean add(E e) {
        linkLast(e);
        return true;
    }
```
添加操作就算在尾节点添加一个节点

```java
/**
    * Links e as last element.
    */
   void linkLast(E e) {
       final Node<E> l = last;
       final Node<E> newNode = new Node<>(l, e, null);
       last = newNode;
       if (l == null)
           first = newNode;
       else
           l.next = newNode;
       size++;
       modCount++;
   }
```
(代码就不解释了,清晰明了)

`remove()` 移走头节点...看了下源码也没发现有什么好说的 :P

注意一下`get(index)` `remove(index)` 都需要遍历部分节点, 而addFirst, remove 等等在两段操作的不需要遍历
`iterator（）上的remove（）` 也不需要,因为他是直接unlink其中一个节点

```java
public E get(int index) {
       checkElementIndex(index);
       return node(index).item;
   }
```

```java
Node<E> node(int index) {
        // assert isElementIndex(index);
        //siez >> 1 是一半
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
