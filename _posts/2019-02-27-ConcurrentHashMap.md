---
layout:     post
title:      ConcurrentHashMap
subtitle:   ConcurrentHashMap
date:       2019-02-27
author:     BY
header-img: img/20190227/20190227.jpg
catalog: true
tags:
    - JAVA面试
    - 基础部分
---
# 前言

>ConcurrentHashMap 同样也分为 1.7 、1.8 版，两者在实现上略有不同。
[《链接1》](https://mp.weixin.qq.com/s?__biz=MzAxNjk4ODE4OQ==&mid=2247484367&idx=1&sn=f2d5d831d4e120f813fa922c194c0f52&chksm=9bed22bdac9aababd95313ec8686d193f77515df50e8bd68c8001a3ca59bc99966118efd9b9f&scene=21#wechat_redirect)
[《链接2》](https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653192083&idx=1&sn=5c4becd5724dd72ad489b9ed466329f5&chksm=8c990d49bbee845f69345e4121888ec967df27988bc66afd984a25331d2f6464a61dc0335a54&scene=21#wechat_redirect)


**Base 1.7**

先来看看 1.7 的实现，下面是他的结构图：

![图1](/img/20190227/2019022701.jpg)

如图所示，是由 `Segment` 数组、`HashEntry `组成，和 `HashMap` 一样，仍然是数组加链表。

它的核心成员变量：
    
    1    /**
    2     * Segment 数组，存放数据时首先需要定位到具体的 Segment 中。
    3     */
    4    final Segment<K,V>[] segments;
    5
    6    transient Set<K> keySet;
    7    transient Set<Map.Entry<K,V>> entrySet;
    
`Segment` 是 `ConcurrentHashMap` 的一个内部类，主要的组成如下：

     1    static final class Segment<K,V> extends ReentrantLock implements Serializable {
     2
     3        private static final long serialVersionUID = 2249069246763182397L;
     4
     5        // 和 HashMap 中的 HashEntry 作用一样，真正存放数据的桶
     6        transient volatile HashEntry<K,V>[] table;
     7
     8        transient int count;
     9
    10        transient int modCount;
    11
    12        transient int threshold;
    13
    14        final float loadFactor;
    15
    16    }
    
看看其中 HashEntry 的组成：

![图2](/img/20190227/2019022702.jpg)


和 `HashMap` 非常类似，唯一的区别就是其中的核心数据如 `value` ，以及链表都是 `volatile` 修饰的，保证了获取时的可见性。

原理上来说：`ConcurrentHashMap` 采用了分段锁技术，其中` Segment` 继承于 `ReentrantLock`。不会像` HashTable` 

那样不管是 `put` 还是 `get` 操作都需要做同步处理，理论上 `ConcurrentHashMap` 支持 `CurrencyLevel` (Segment 数组数量)的线程并发。

每当一个线程占用锁访问一个 `Segment `时，不会影响到其他的 `Segmen`t。

下面也来看看核心的 put get 方法。

**put 方法**

     1    public V put(K key, V value) {
     2        Segment<K,V> s;
     3        if (value == null)
     4            throw new NullPointerException();
     5        int hash = hash(key);
     6        int j = (hash >>> segmentShift) & segmentMask;
     7        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
     8             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
     9            s = ensureSegment(j);
    10        return s.put(key, hash, value, false);
    11    }
    
首先是通过 key 定位到 Segment，之后在对应的 Segment 中进行具体的 put。

     1        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
     2            HashEntry<K,V> node = tryLock() ? null :
     3                scanAndLockForPut(key, hash, value);
     4            V oldValue;
     5            try {
     6                HashEntry<K,V>[] tab = table;
     7                int index = (tab.length - 1) & hash;
     8                HashEntry<K,V> first = entryAt(tab, index);
     9                for (HashEntry<K,V> e = first;;) {
    10                    if (e != null) {
    11                        K k;
    12                        if ((k = e.key) == key ||
    13                            (e.hash == hash && key.equals(k))) {
    14                            oldValue = e.value;
    15                            if (!onlyIfAbsent) {
    16                                e.value = value;
    17                                ++modCount;
    18                            }
    19                            break;
    20                        }
    21                        e = e.next;
    22                    }
    23                    else {
    24                        if (node != null)
    25                            node.setNext(first);
    26                        else
    27                            node = new HashEntry<K,V>(hash, key, value, first);
    28                        int c = count + 1;
    29                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
    30                            rehash(node);
    31                        else
    32                            setEntryAt(tab, index, node);
    33                        ++modCount;
    34                        count = c;
    35                        oldValue = null;
    36                        break;
    37                    }
    38                }
    39            } finally {
    40                unlock();
    41            }
    42            return oldValue;
    43        }
    
虽然 `HashEntry `中的 `value` 是用 `volatile` 关键词修饰的，但是并不能保证并发的原子性，所以 `put`操作时仍然需要加锁处理。

首先第一步的时候会尝试获取锁，如果获取失败肯定就有其他线程存在竞争，则利用 scanAndLockForPut() 自旋获取锁。

![图3](/img/20190227/2019022703.jpg)


尝试自旋获取锁。

如果重试的次数达到了 `MAX_SCAN_RETRIES `则改为阻塞锁获取，保证能获取成功。

![图4](/img/20190227/2019022704.jpg)


再结合图看看 put 的流程。

将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry。

遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。

不为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。

最后会解除在 1 中所获取当前 Segment 的锁。

**get 方法**

     1    public V get(Object key) {
     2        Segment<K,V> s; // manually integrate access methods to reduce overhead
     3        HashEntry<K,V>[] tab;
     4        int h = hash(key);
     5        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
     6        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
     7            (tab = s.table) != null) {
     8            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
     9                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
    10                 e != null; e = e.next) {
    11                K k;
    12                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
    13                    return e.value;
    14            }
    15        }
    16        return null;
    17    }
    
get 逻辑比较简单：

只需要将 Key 通过 Hash 之后定位到具体的 Segment ，再通过一次 Hash 定位到具体的元素上。

由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。

ConcurrentHashMap 的 get 方法是非常高效的，因为整个过程都不需要加锁.

**Base 1.8**

1.7 已经解决了并发问题，并且能支持 N 个 Segment 这么多次数的并发，但依然存在 HashMap 在 1.7 版本中的问题。

那就是查询遍历链表效率太低。

因此 1.8 做了一些数据结构上的调整。

首先来看下底层的组成结构：

![图5](/img/20190227/2019022705.jpg)


看起来是不是和 1.8 HashMap 结构类似？

其中抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性。

![图6](/img/20190227/2019022706.jpg)

也将 1.7 中存放数据的 HashEntry 改为 Node，但作用都是相同的。

其中的 val next 都用了 volatile 修饰，保证了可见性。

**put 方法**

![图7](/img/20190227/2019022707.jpg)

重点来看看 put 函数：


根据 key 计算出 hashcode 。

判断是否需要进行初始化。

f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。

如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。

如果都不满足，则利用 synchronized 锁写入数据。

如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。

**get 方法**

![图8](/img/20190227/2019022708.jpg)

根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。

如果是红黑树那就按照树的方式获取值。

就不满足那就按照链表的方式遍历获取值。

1.8 在 1.7 的数据结构上做了大的改动，采用红黑树之后可以保证查询效率（O(logn)），

甚至取消了 ReentrantLock 改为了 synchronized，这样可以看出在新版的 JDK 中对 synchronized 优化是很到位的。

