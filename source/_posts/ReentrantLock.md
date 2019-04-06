title: ReentrantLock
author: alben.wong
abbrlink: '6e505175'
tags:
  - reentrantlock
  - juc
categories:
  - java
  - juc
date: 2019-04-02 20:23:00
keywords: reentrantlock 原理 aqs
description: ReentrantLock顾名思义是可重入锁，提供了公平性的机制，内部基于AbstractQueuedSynchronizer来实现。
---
## 概要
ReentrantLock顾名思义是可重入锁，提供了公平性的机制，内部基于AbstractQueuedSynchronizer来实现。
所以ReentrantLock只是AbstractQueuedSynchronizer的一个使用场景的实现。

## 使用
ReentrantLock的使用相信大家都很熟悉，主要是以下几个方法
```java
//阻塞程序，直到成功的获取锁
public void lock();
//与lock()不同的地方是，它可以响应程序中断，如果被其他程序中断了，则抛出InterruptedException。
public void lockInterruptibly() throws InterruptedException;
//尝试获取锁，该方法会立即返回，并不会阻塞程序。如果获取锁成功则返回true，反之则返回false。
public boolean tryLock();
//尝试获取锁，如果能获取锁则直接返回true；否则阻塞等待，阻塞时长由传入的参数来决定，在等待的同时响应程序中断，如果发生了中断则抛出InterruptedException；如果在等待的时间中获取了锁则返回true，反之返回false。
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
//释放锁
public void unlock();
//新建一个条件，一个Lock可以关联多个条件。
public Condition newCondition();
```

## 实现原理
ReentrantLock的实现是基于AbstractQueuedSynchronizer的，如果不清楚可以看{% post_link AQS AQS的实现 %}。
如果你已经大概了解AQS的实现，那么了解ReentrantLock的实现对于你来说就会显得非常简单了。

ReentrantLock持有一个内部类对象`Sync`的实例，上面的`lock()`、`unlock()`等方法都是调用到sync的方法的
```java
    private final Sync sync;

    public void lock() {
        sync.lock();
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    public void unlock() {
        sync.release(1);
    }
```

### Sync的实现
所以我们重点看看`Sync`的实现
```java
    //很重要的一点，继承了AbstractQueuedSynchronizer
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        //lock方法由NonfairSync和FairSync实现
        abstract void lock();

        //这里实现了一个不公平的的TryAcquire，放在Sync这个类的原因是，无论公平锁还是非公平锁，都会用到这个方法
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取aqs的state状态，用来标记锁是否被获取了
            int c = getState();
            //等于0说明没有线程占用锁，然后通过cas尝试竞争一下锁，其实就是调用aqs的compareAndSetState方法，把state设置为1，如果成功，则标记当前线程获得锁
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果state != 0，还有一种情况就是当前线程自己之前已经获得锁了，这次是再次进来临界区，所以要看看exclusiveOwnerThread是不是等于自己，如果是则state++。所以如果是重入锁的情况，会出现 state > 1。
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //否则就是获得锁失败
            return false;
        }

        //释放锁，因为释放锁的操作都是一样，所以放在Sync统一处理
        protected final boolean tryRelease(int releases) {
            //state --
            int c = getState() - releases;
            //防止释放锁的线程不是获得锁的线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //如果state == 0，如果锁被释放掉了。如果 state != 0，说明锁在被重入的情况
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

    }
```

### 公平/非公平锁的实现
ReentrantLock是提供锁的公平机制的，锁是否公平定义如下
公平锁：新线程进来临界区争夺资源，如果之前有线程在排队，那么它必须紧接着排在队列后面
不公平锁：新线程可以跟已经在排队的线程一起竞争，如果失败也是乖乖排队

ReentrantLock在`Sync`的基础上实现了非公平的`NonfairSync`和公平的`FairSync`
看看ReentrantLock的初始化
```java
    //默认使用非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    //通过带参数的构造函数决定使用公平锁还是非公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

#### NonfairSync
```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        //先去直接cas一下获得锁，碰一下运气，失败则调用nonfairTryAcquire方法，虽然失败了，但有可能是重入的情况，如果都不是，则乖乖排队把
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
            //aqs的acquire方法，里面会调用下面的tryAcquire方法，熟悉aqs大概知道，如果tryAcquire返回true则获得锁成功，如果失败则进入CLH队列排队去。
                acquire(1);
        }

        //aqs内部会调用这个方法
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

#### FairSync
```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
        //直接调用aqs的acquire
        //可以跟上面的NonfairSync的lock方法对比一下有啥不同
        final void lock() {
            acquire(1);
        }

        //aqs内部会调用这个方法
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //hasQueuedPredecessors是aqs自带的方法，表示看看AQS的队列有没有节点在排队，用来快速判断是否有别的线程在排队
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //重入处理，跟上面的一样
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

```

#### 对比
其实可以看出公平锁和非公平锁的差别在于，非公平锁在尝试获得锁的第一时间去尝试获得锁，这时如果锁没被占有，那么它将会跟排队的第一节节点进行竞争。
非公平锁：效率和性能比较高，适于用有TPS要求的场景，但是会出现“饥饿”问题，即在排队的线程等了很久都没有获得锁
公平锁：保证FIFO，不会出现“饥饿”问题
ReentrantLock默认是非公平锁。

### 重入的实现
这个已经在上面已经提到过了
```java
            //重入处理
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
```
在获得锁的同时会记录锁的持有者`exclusiveOwnerThread`，持有者再次获得锁时则可以不用竞争排队，无条件获得锁，不过锁的状态`state`会+1，用来记录重入的次数，因为在`unlock()`方法会把`state`减一，完全释放锁时`state == 0`。

## 与synchronized区别
ReentrantLock的功能与`synchronized`类似，经常拿来比较
### 相同点
1. 互斥、阻塞型的
即只有一个线程获得锁，其他线程必须在阻塞等待
2. 可重入性
同一个线程可以再次获得锁，无需等待

### 不同点
相比来说，`ReentrantLock`显得更加“高级”一点。synchronized是Java的关键字，由JVM实现其功能，在字节码插入monitorenter指令和monitorexit指令。ReentrantLock则直接由Java代码实现。synchronized更加底层。
1. 持有的对象监视器不同
2. ReentrantLock可中断
3. ReentrantLock提供公平机制
4. ReentrantLock可以绑定多个条件，即Condition对象，提供更加丰富的多线程并发骚操作。 

JDK1.6之后synchronize在语义上很清晰，进行很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等，在性能上并不比`Lock`差。

综上，在性能差别不大的情况，根据自己的需求来选择，如果对锁的要求简单的话，可以直接用synchronized，如果复杂同步操作，则选择ReentrantLock。

## 总结
本文介绍基于AbstractQueuedSynchronizer的ReentrantLock的实现，通过源码可以看出重入性和公平性的实现，并对比了与synchronized的区别，阅读本文后希望大家对ReentrantLock的使用更加了然于心。