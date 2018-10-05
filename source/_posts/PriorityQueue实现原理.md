title: PriorityQueue实现原理
author: alben.wong
tags:
  - PriorityQueue
  - 优先级队列
  - 数据结构
categories:
  - java
abbrlink: 49dd368f
keywords: PriorityQueue 优先级队列
description: PriorityQueue是一个重要数据结构，是DelayQueue的底层实现，为例如任务调度的实现提供底层的数据结构。
date: 2018-10-05 14:40:00
---
## 概要
PriorityQueue是一个重要数据结构，是DelayQueue的底层实现，为例如任务调度的实现提供底层的数据结构。

## PriorityQueue原理分析

如果你了解堆排序的话，它的实现对你而言就显得很简单。

先看看PriorityQueue有什么属性
```java
/**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    transient Object[] queue; // non-private to simplify nested class access

    /**
     * The number of elements in the priority queue.
     */
    private int size = 0;

    /**
     * The comparator, or null if priority queue uses elements'
     * natural ordering.
     */
    private final Comparator<? super E> comparator;
```
主要属性有 queue 一个数组，Comparator 比较器。

添加元素的 add/offer 方法：
```java
/**
     * Inserts the specified element into this priority queue.
     *
     * @return {@code true} (as specified by {@link Queue#offer})
     * @throws ClassCastException if the specified element cannot be
     *         compared with elements currently in this priority queue
     *         according to the priority queue's ordering
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
```
加入元素，grow是扩容，元素的插入主要看siftUp。

先看看grow方法：
```java
/**
     * Increases the capacity of the array.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity);
    }
```
奇怪的 grow，就如注释所说的，小于 64 时扩容 2 倍，大于 64 时扩容 50%。

siftUp方法：
```java
/**
     * Inserts item x at position k, maintaining heap invariant by
     * promoting x up the tree until it is greater than or equal to
     * its parent, or is the root.
     *
     * To simplify and speed up coercions and comparisons. the
     * Comparable and Comparator versions are separated into different
     * methods that are otherwise identical. (Similarly for siftDown.)
     *
     * @param k the position to fill
     * @param x the item to insert
     */
    private void siftUp(int k, E x) {
        if (comparator != null)
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```
siftUp 方法，为什么叫 up 呢，因为插入的位置是数组的最后，也就是二叉树的最后一个节点，所以要向上调整，这里就涉及堆排序的调整。  
comparator 为空时，用 compareTo 比较，直到 parent 比插入的元素大，否则交换，就是这么简单。


取出元素的poll方法：
```java
public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);
        return result;
    }
```

取出元素，然后把最后的元素放在第一个的位置，然后调整，本质也是涉及堆排序的调整。


## 总结
利用堆排序实现插入、取出等操作时的重排序，目的是效率较高（我一开始认为是一个线性 array，然后通过 shift - 类似插入排序的样子，但是想想这是个 O(n) 的操作，堆排序是 O(logn)快多了）；  
默认实现是最小堆，当然也可以传入相反比较结果的 Comparator 实现最大堆；  
没有传入 Comparator 的话，默认使用元素的 compareTo 方法，所以元素要是 Comparable 的，不然会报错；  
全程没有锁，不支持高并发，不过有PriorityBlockingQueue；  
不允许null元素；  


## 参考资料
https://www.cnblogs.com/CarpenterLee/p/5488070.html


