---
layout:     post
title:      Hashmap，Hashtable，Concurrenthashmap
subtitle:   一文读懂JDK7,8,JD9的hashmap，hashtable，concurrenthashmap及他们的区别
date:       2019-02-25
author:     BY
header-img: img/20190225/20190225.jpg
catalog: true
tags:
    - JAVA面试
    - 基础部分
---
# 前言

>一文读懂JDK7,8,JD9的hashmap，hashtable，concurrenthashmap及他们的区别[《链接》](https://mp.weixin.qq.com/s/_I7XvwCUiXXWNhUn_tVgQg).



## 1：hashmap简介（如下，数组-链表形式）

**HashMap的存储结构**

![图1](/img/20190225/2019022501.jpg)

图中，紫色部分即代表哈希表，也称为哈希数组（默认数组大小是16，每对key-value键值对其实是存在map的内部类entry里的），

数组的每个元素都是一个单链表的头节点，跟着的绿色链表是用来解决冲突的，如果不同的key映射到了数组的同一位置处，

就会采用头插法将其放入单链表中。

## 2：hashmap原理（即put和get原理）

### 2.1 put原理

1. 根据key获取对应hash值：int hash = hash（key.hash.hashcode（））

2. 根据hash值和数组长度确定对应数组引int i = indexFor(hash, table.length); 简单理解就是i = hash值%模以 

数组长度（其实是按位与运算）。如果不同的key都映射到了数组的同一位置处，就将其放入单链表中。且新来的是放在头节点。

### 2.2 get原理

1. 通过hash获得对应数组位置，遍历该数组所在链表（key.equals（））

## 3.1：hashcode相同，冲突怎么办？

“头插法”，放到对应的链表的头部。

## 3.2：为什么是头插法（为什么这么设计）？

因为HashMap的发明者认为，后插入的Entry被查找的可能性更大，所以放在头部（因为get（）查询的时候会遍历整个链表）。

## 4.1：hashmap的默认数组长度是多少？

默认为16

## 4.2：为什么？

之所以选择16，是为了服务于从key映射到index的hash算法（看下面）。

## 5.1：hashmap达到默认负载因子（0.75）怎么办？

自动双倍扩容，扩容后重新计算每个键值对位置。且长度必须为16或者2的幂次

## 5.2：为啥要16或者2的幂次？

若不是16或者2的幂次，位运算的结果不够均匀分布，显然不符合Hash算法均匀分布的原则。

反观长度16或者其他2的幂，Length-1的值是所有二进制位全为1，这种情况下，index的结果等同于HashCode后几位的值。

只要输入的HashCode本身分布均匀，Hash算法的结果就是均匀的。

## 6.1：hashmap是线程安全的吗？

不是。

## 6.2 ：为什么？

因为没加锁

## 6.3： 那在并发时会导致什么问题？

hashmap在接近临界点时，若此时两个或者多个线程进行put操作，都会进行resize（扩容）和ReHash（为key重新计算所在位置），

而ReHash在并发的情况下可能会形成链表环。在执行get的时候，会触发死循环，引起CPU的100%问题。



_注：jdk8已经修复hashmap这个问题了，jdk8中扩容时保持了原来链表中的顺序。但是HashMap仍是非并发安全，

在并发下，还是要使用ConcurrentHashMap。_

## 6.4： 如何判断有环形表？

最优：首先创建两个指针A和B（在java里就是两个对象引用），同时指向这个链表的头节点。然后开始一个大循环，

在循环体中，让指针A每次向下移动一个节点，让指针B每次向下移动两个节点，然后比较两个指针指向的节点是否相同。

如果相同，则判断出链表有环，如果不同，则继续下一次循环。

理解例子：在一个环形跑道上，两个运动员在同一地点起跑，一个运动员速度快，一个运动员速度慢。

当两人跑了一段时间，速度快的运动员必然会从速度慢的运动员身后再次追上并超过，原因很简单，因为跑道是环形的。

![图2](/img/20190225/2019022502.jpg)

##　7： hashmap  和  hashtable  区别？

两者的区别线程效率数组默认值null值hashmap不安全更高16key-value都允许hashtable安全略低11不允许（抛异常）

## 8.0：那hashmap不安全，hashtable性能又低，怎么办？

用concurrenthashmap，即保证安全，性能又可以保证。

## 8.1：那concurrenthashmap究竟是什么？

整个ConcurrentHashMap的结构如下：

![图3](/img/20190225/2019022503.jpg)


理解：hashmap是有entry数组组成，而concurrenthashmap则是Segment数组组成。而Segment又是什么呢？

Segment本身就相当于一个HashMap。

同HashMap一样，Segment包含一个HashEntry数组，数组中的每一个HashEntry既是一个键值对，也是一个链表的头节点。

单一的Segment结构如下（是不是看着就是hashmap）：

![图4](/img/20190225/2019022504.jpg)


像这样的Segment对象，在ConcurrentHashMap集合中有多少个呢？有2的N次方个，共同保存在一个名为segments的数组当中。

可以说，ConcurrentHashMap是一个二级哈希表。在一个总的哈希表下面，有若干个子哈希表。（这样类比理解多个hashmap组成一个cmap）

## 8.2：那他的put和get方法呢？

**Put方法：**

1. 为输入的Key做Hash运算，得到hash值。

2. 通过hash值，定位到对应的Segment对象

3. 获取可重入锁

4. 再次通过hash值，定位到Segment当中数组的具体位置。

5. 插入或覆盖HashEntry对象。

6. 释放锁。

**Get方法：**

1. 为输入的Key做Hash运算，得到hash值。

2. 通过hash值，定位到对应的Segment对象

3. 再次通过hash值，定位到Segment当中数组的具体位置。

由此可见，和hashmap相比，ConcurrentHashMap在读写的时候都需要进行二次定位。先定位到Segment，再定位到Segment内的具体数组下标。

## 9：     hashmap  和     concurrenthashmap区别？

线程：   不安全                   安全

## 10.1：为啥concurrenthashmap和hashtable都是线程安全，但是前者性能更高

因为前者是用的分段锁，根据hash值锁住对应Segment对象，当hash值不同时，使其能实现并行插入，效率更高，而hashtable则会锁住整个map。

如何理解并行插入：当cmap需要put元素的时候，并不是对整个map进行加锁，

而是先通过hashcode来知道他要放在那一个分段（Segment对象）中，然后对这个分段进行加锁，所以当多线程put的时候，

只要不是放在同一个分段中，就实现了真正的并行的插入。

但是，在统计size的时候，就是获取concurrenthashmap全局信息的时候，就需要获取所有的分段锁才能统计（即效率稍低）。

## 10.2：分段锁的设计解决的是什么问题？

分段锁的设计目的是细化锁的粒度，当操作不需要更新整个数组的时候，就仅仅针对数组中的一部分行加锁操作。

## 11：JDK1.7的hashmap和JDK1.8的hashmap的区别（即1.8做了哪些优化）？

1. 为了加快查询效率，java8的hashmap引入了红黑树结构，当数组长度大于默认阈值64时，且当某一链表的元素>8时，

该链表就会转成红黑树结构，查询效率更高。（问题来了，什么是红黑树？什么是B+树？（mysql索引有B+树索引）什么是B树？什么是二叉查找树？）

数据结构方面的知识点会更新在【数据结构专题】，这里不展开。

这里只简单的介绍一下红黑树：

红黑树是一种自平衡二叉树，拥有优秀的查询和插入/删除性能，广泛应用于关联数组。

对比AVL树，AVL要求每个结点的左右子树的高度之差的绝对值（平衡因子）最多为1，

而红黑树通过适当的放低该条件（红黑树限制从根到叶子的最长的可能路径不多于最短的可能路径的两倍长，结果是这个树大致上是平衡的），

以此来减少插入/删除时的平衡调整耗时，从而获取更好的性能，而这虽然会导致红黑树的查询会比AVL稍慢，

但相比插入/删除时获取的时间，这个付出在大多数情况下显然是值得的。好了我知道你们看晕了，移步去看看我的【数据结构专题】吧。

2. 优化扩容方法，在扩容时保持了原来链表中的顺序，避免出现死循环

## 12：JDK1.7的concurrenthashmap和JDK1.8又有什么区别？

1.8的实现已经抛弃了Segment分段锁机制，利用Node数组+CAS+Synchronized来保证并发更新的安全，底层采用数组+链表+红黑树的存储结构。

![图5](/img/20190225/2019022505.jpg)

java给我们带来了并发安全的ConcurrentHashMap，它的实现是依赖于 Java 内存模型，

所以我们在了解 ConcurrentHashMap 的之前必须了解一些底层的知识：

* java内存模型

* java中的Unsafe

* java中的CAS

* java同步器AQS

* ReentrantLock

所以在这里我不准备深入讲解ConcurrentHashMap ，我会在【并发编程】专题通过一步步详解并发基础，

从java内存模型，synchronized，volatile，Unsafe到CAS，AQS，各种锁再到JUC并发包相关。

先放张java内存模型的思维导图勾引一波，光java内存模型一个点就有这么多要讲的了。



## 13：那么问题来了，什么是CAS？

关于CAS方面的知识点，又会涉及到ABA问题，又可以扯到乐观锁悲观锁，锁编程，AQS等，相关内容将更新在【并发编程专题】，这里不做展开

![图6](/img/20190225/2019022506.png)