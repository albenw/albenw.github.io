title: Striped64和LongAdder
author: alben.wong
abbrlink: 6518ba8
tags:
  - juc
  - longadder
categories:
  - java
  - juc
keywords: Striped64 LongAdder
description: LongAdder是JDK1.8新增的，并发性、性能比AtomicLong高的，用于计数的这么一个类。本文着重讲解LongAdder的设计思想。
date: 2019-10-08 23:23:00
---
## 概要

LongAdder是JDK1.8新增的，并发性、性能比AtomicLong高的，用于计数的这么一个类。本文着重讲解LongAdder的设计思想。



## 背景

AtomicLong的实现方式是内部有个value 变量，当多线程并发自增，自减时，均通过CAS 指令从机器指令级别操作保证并发的原子性。唯一会制约AtomicLong高效的原因是高并发，高并发意味着CAS的失败几率更高， 重试次数更多，越多线程重试，CAS失败几率又越高，变成恶性循环，AtomicLong效率降低。



## 设计思想

而LongAdder将把一个value拆分成若干cell，把所有cell加起来，就是value。所以对LongAdder进行加减操作，只需要对不同的cell来操作，不同的线程对不同的cell进行CAS操作，CAS的成功率当然高了（试想一下3+2+1=6，一个线程3+1，另一个线程2+1，最后是8，LongAdder没有乘法除法的API）。



可是在并发数不是很高的情况，拆分成若干的cell，还需要维护cell和求和，效率不如AtomicLong的实现。LongAdder用了巧妙的办法来解决了这个问题。

初始情况，LongAdder与AtomicLong是相同的，只有在CAS失败时，才会将value拆分成cell，每失败一次，都会增加cell的数量，这样在低并发时，同样高效，在高并发时，这种“自适应”的处理方式，达到一定cell数量后，CAS将不会失败，效率大大提高。

LongAdder是一种以空间换时间的策略。

![upload successful](/images/Striped64和LongAdder__0.png)



## 总结

1. 最重要的是设计思想，采用热点分离的方法，把线程的竞争分散到Cell中，提高了并发能力。即把原来只有单个资源的竞争分散为多个，这样就可以让多个线程并发操作，不必阻塞或CAS重试。

2. 流程大概是有一个base字段，当不存在竞争时往base累计，存在竞争时把累计分散到Cell数组中，不过sum时要把base和Cell数组全部累加在一起。

3. LongAdder是Striped64的实现，Striped64实现了主要的核心功能如longAccumulate；而LongAdder对外提供了接口，并补充、扩展了一些方法，如increment、sum、reset等等。

4. Cell类使用了@sun.misc.Contended来解决内存伪共享，因为在数组中，内存是连续分配的，伪共享尤为明显。


