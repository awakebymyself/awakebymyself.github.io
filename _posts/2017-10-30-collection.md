---
layout: post
title:  "Java集合框架源码分析(一) ArrayList"
categories: Collection
tags:  集合
author: Lzg
---


*今天开始把源码分析的心得记录下,因为阅读过之后很容易忘记!
也方便自己以后方便查阅或者复习!*

***

`OK, 废话不多说, 今天要说的就是ArrayList`

Java 集合最顶端的接口是`Collection`, 下面分为子接口`List`
和`Set`.  
List 可以直接说成是表, 那么ArrayList字面意思就是有数组组成的表
*先说下表的特点,是读取的速度快,根据下边读取/set速度很快, 但是插入删除则需要复制数组,性能会变差*

**首先代码里面有一个很关键的transient 变量`modCount` 用来记录数组的修改**

*ArrayList如何扩容的?*
  每次在向List 添加(add)元素的时候首先会去确保容量是足够的,
  确保最小的容量, ArrayList可以存储null!
  `java`

    private void ensureExplicitCapacity(int minCapacity) {
       modCount++;

       // overflow-conscious code
       if (minCapacity - elementData.length > 0)
           grow(minCapacity);
         }

再来看下grow代码的内容*扩容策略是增加50%, 默认第一次插入数值的时候大小是10, 最好指定容量大小
 这样避免发生system.arraycopy*

```java
private void grow(int minCapacity) {
       // overflow-conscious code
       int oldCapacity = elementData.length;
       int newCapacity = oldCapacity + (oldCapacity >> 1);
       if (newCapacity - minCapacity < 0)
           newCapacity = minCapacity;
       if (newCapacity - MAX_ARRAY_SIZE > 0)
           newCapacity = hugeCapacity(minCapacity);
       // minCapacity is usually close to size, so this is a win:
       elementData = Arrays.copyOf(elementData, newCapacity);
   }
```

  这段逻辑很简单, 主要就是判断当前的数组长度加上右移的数值 是否小于传入的最小的容量.
  然后求出需要的新的数组容量,并最后调用`System.arraycopy` 复制数组!


`size` 表示这个表的大小,有多少个数值,不要和length搞混, 他们并不一定相等!

`add`想数组末尾添加元素

**如何在ArrayList删除元素**

  * 用增强for循环 会引发ConcurrentModificationException，因为元素在使用的时候发生了并发的修改，导致异常抛出。
  ```java
  for(String x:list){
      if(x.equals("del"))
          list.remove(x);
  }
```
  * 用for i 遍历的方式,可以删除不报错,但会产生问题,比如说删除第一个元素的时候,剩下的元素索引都将向前移动一位,在访问的时候就会产生问题
  ```java
  for(int i=0;i<list.size();i++){
      if(list.get(i).equals("del"))
          list.remove(i);
  }
```
  * 正确的方式是使用迭代器! 并且用`iterator.remove()!`
  ```java
  Iterator<String> it = list.iterator();
while(it.hasNext()){
    String x = it.next();
    if(x.equals("del")){
        it.remove();
    }
}
```

在源码里面操作数组的时候有一个方法很重要`System.arraycopy `这个方法对于数组是引用类型的来说
是*浅拷贝*!
`Objects.clone` 是浅拷贝, 也就是说对于对象里面存在着引用类型的,需要我们手动调用他的clone方法!

`还有一种深拷贝的方式是进行序列化以及反序列化`, 下面是两种不同的实现!
* 自己重新实现Object.clone() ,需要实现Cloneable接口,并且对象中的每个引用类型都得
自己重新clone方法! 非常繁琐容易出错
```java
public class User implements Cloneable{
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    private Object clone2() throws CloneNotSupportedException {
        User clone = (User) super.clone();
        clone.setHelper((Helper) clone.getHelper().clone());
        return clone;
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        User  user = new User( 24, "jack",new Helper("helper"));

        User clone = (User) user.clone();
        System.out.println(user == clone);//false
        System.out.println(user.getClass() == user.getClass()); //true

        clone.getHelper().name = "helper2";
        System.out.println(user.getHelper().name);// helper2  shallow copy

        User clone2 = (User) user.clone2();
        clone2.getHelper().name ="xxxxx";
        System.out.println(user.getHelper().name); // helper2 deep copy
    }

    static class Helper implements Cloneable {
        private String name;

        public Helper(String name) {
            this.name = name;
        }

        @Override
        protected Object clone() throws CloneNotSupportedException {
            return super.clone();
        }
    }
}
```

 *方法二:  用序列化的方式! 不过这样里面的每个引用类型都必须确保实现序列化接口!!*
```java
private static User deepClone(User user) {
       User user1 = null;
       // Write the object out to a byte array
       ByteArrayOutputStream arrayStream = new ByteArrayOutputStream();
       try {
           ObjectOutputStream out = new ObjectOutputStream(arrayStream);
           out.writeObject(user);

           out.flush();
           out.close();

           // Make an input stream from the byte array and read
           // a copy of the object back in.
           ObjectInputStream in = new ObjectInputStream(
                   new ByteArrayInputStream(arrayStream.toByteArray()));
            user1 = (User) in.readObject();
       } catch (IOException e) {
           e.printStackTrace();
       } catch (ClassNotFoundException e) {
           e.printStackTrace();
       }
       return user1;
   }
```



在这边插一句, 对于数组的拷贝来说, 如果数组类型是引用类型,那么是浅拷贝!
```java
int[] ints = {1,2};
      User[] users = {new User("jack"), new User("rose")};

      int[] ints1 = ints.clone();
      User[] users1 = users.clone();
      System.out.println(ints == ints1);//false
      System.out.println(users == users1);//false

      ints1[0] = 10;
      System.out.println(ints[0]);//1

      users[0].name = "tom";
      System.out.println(users[0].getName());//tom
```

*如何实现引用数组类型的深拷贝呢?*

```java
User[] old = {new User("jack"), new User("rose")};
       User[] newUser = new User[3];
       for (int i = 0; i < old.length; i++) {
           newUser[i] = new User(old[i].name);
       }

   }
```


## CopyOnWriteArrayList

`源码还没看,未完待续...`
