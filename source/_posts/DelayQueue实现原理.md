title: DelayQueue实现原理
author: alben.wong
abbrlink: 255bd548
tags:
  - DelayQueue
  - 延迟队列
categories:
  - java
keywords: DelayQueue 延迟队列
description: 任务调度和缓存框架的都会用到DelayQueu作为底层实现，了解它可以让我们更好理解这些框架的本质。
date: 2018-10-05 20:53:00
---
## 概要
任务调度和缓存框架的都会用到DelayQueu作为底层实现，了解它可以让我们更好理解这些框架的本质。

## 背景
如果要判断一个缓存对象超时没有，一种笨笨的办法就是，使用一个后台线程，遍历所有对象，挨个检查。这种笨笨的办法简单好用，但是对象数量过多时，可能存在性能问题，检查间隔时间不好设置，间隔时间过大，影响精确度，多小则存在效率问题。而且做不到按超时的时间顺序处理。  
那么DelayQueue就是用来解决这类问题。

## 实现原理
如果看过 PriorityQueue  的源码，就会发现 DelayQueue 的源码实现起来很简答，基本都是调用 PriorityQueue 的插入和取出。
不过 DelayQueue 支持高并发，即每个方法开头和结尾都有用 ReentrantLock。

### take方法

我们重点来看看 take 方法：
```java
/**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```
可以看出延迟的实现原理就是用到了 Condition.awaitNanos(delay) 方法。
先 peek 看看有没有元素，再看看元素有没有过期，过期就 poll 取出，还没过期就是 await 等待。
这里有两点需要注意：

#### leader线程的作用 
先看看官方注释：
```java
 /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader = null;
```
说了是用到 Leader-Follower 模式。  
如果一个线程是 leader 线程，那么它只会等待 available.awaitNanos(delay) 这么多时间，其他后来的 follower 线程只能干等。  
意思就是一定是 leader 线程先取到头元素，其他线程需要等待 leader 线程的唤醒。  
这样就可以简化竞争的操作，直接让后面的线程等待，把竞争交给 Condition 来做。 

#### first == null
目的是为了做 GC。假设没有这一句，那么这里很有可能是 follower 线程在等待的过程中一直持有 first 的引用，而 leader 线程已经完成任务了，都把 first 都释放了，原来希望被回收的 first 却一直没有被回收。在极端的情况下，在一瞬间高并发，会有大量的 follower 线程持有 first，而需要等这些线程都会唤醒后，first 才会被释放回收。



### offer方法

offer 方法，add 和 put 最终还是调到 offer 方法。

```java
/**
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

放入元素，如果插入的元素是放在了头部的话：  
1. 把 leader 线程置为 null   
因为 leader 的意义就是想要取头元素的那个线程，那么旧的 leader 将没有意义。  

2. 唤醒在等待的线程  
原本线程都在等待头元素，但是头元素改变了，就唤醒一个线程让它重新取出头元素，并成为新的 leader （看 take 方法里面是一个 for 的死循环）。



## 总结

- 无界队列 - 因为本质是PriorityQueue，PriorityQueue会无限扩展；  
- item 需要实现 Delayed 接口，实现 compareTo 和 getDelay 方法，前者用于放入队列时排序，后者用于如果返回小于 0 且在队列头，则可以取出来；  
- 注意 getDelay 返回的是 NANOSECONDS；  
- poll 头元素还没过期则会返回 null；  
- 重入锁是非公平的；  
- 是实现定时任务的关键；  
- 关于 compareTo 和 getDelay，我之前有点混淆，compareTo 是决定放到队列的位置，getDelay 是觉得取出来时的延迟时间；compareTo 和 getDelay 是没有关系的，就是说，队列头的元素可能 getDelay 很大，它后面的元素 getDelay 很小，不一定是说 getDelay 小是放在队列前面的；一般实际使用，我们会用使用相同的属性来做 compareTo 和 getDelay，使到它们是一致的。  


## 参考资料
http://cmsblogs.com/?p=2413
https://www.zybuluo.com/mikumikulch/note/712598