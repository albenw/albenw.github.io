title: ReentrantReadWriteLock
author: alben.wong
abbrlink: f8dba914
tags:
  - juc
  - reentrantreadwritelock
categories:
  - java
  - juc
keywords: reentrantreadwritelock 读写锁 原理
description: ReentrantReadWriteLock利用读写分离的思想，在读多写少的情况下有更好的性能表现。
date: 2019-04-07 22:09:00
---
## 概要
ReentrantReadWriteLock顾名思义"可重入的读写锁"，它跟ReentrantLock有点类似。ReentrantReadWriteLock主要是利用读写分离的思想，读取数据使用“读锁” 实现多个读线程可以并行读；写数据使用“写锁” 实现只能由一个线程写入。相对于ReentrantLock，在读多写少的情况下，使用ReentrantReadWriteLock会有更好的性能表现。
本文将介绍ReentrantReadWriteLock的使用及实现原理。

## 应用场景
ReentrantReadWriteLock的使用场景就是在读多写少情况下以提高性能。看看官方文档的描述以及例子:
> ReentrantReadWriteLocks can be used to improve concurrency in some
uses of some kinds of Collections. This is typically worthwhile
only when the collections are expected to be large, accessed by
more reader threads than writer threads, and entail operations with
overhead that outweighs synchronization overhead. For example, here
is a class using a TreeMap that is expected to be large and concurrently accessed.

对TreeMap加读写锁例子:
```java
 class RWDictionary {
   private final Map<String, Data> m = new TreeMap<String, Data>();
   private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
   private final Lock r = rwl.readLock();
   private final Lock w = rwl.writeLock();

   public Data get(String key) {
     r.lock();
     try { return m.get(key); }
     finally { r.unlock(); }
   }

   public String[] allKeys() {
     r.lock();
     try { return m.keySet().toArray(); }
     finally { r.unlock(); }
   }

   public Data put(String key, Data value) {
     w.lock();
     try { return m.put(key, value); }
     finally { w.unlock(); }
   }

   public void clear() {
     w.lock();
     try { m.clear(); }
     finally { w.unlock(); }
   }
```

## 实现原理
对ReentrantReadWriteLock有了一个感性的认识后，我们看看它的实现原理。
首先ReentrantReadWriteLock是基于`AQS`实现的，如果你对AQS还不是很了解，建议你先看看{% post_link AQS AQS %}。

### 读写/内部状态
在了解到ReentrantReadWriteLock后，你是否有个疑问，它是怎么区分读锁和写锁的？
像ReentrantLock一样，ReentrantReadWriteLock内部有一个`Sync`类，它继承`AbstractQueuedSynchronizer`，我们知道AQS内部只有一个int类型的`state`变量来表示状态。那么ReentrantReadWriteLock.Sync是怎么做的呢？
```java
  /*
    * Read vs write count extraction constants and functions.
    * Lock state is logically divided into two unsigned shorts:
    * The lower one representing the exclusive (writer) lock hold count,
    * and the upper the shared (reader) hold count.
    */

  static final int SHARED_SHIFT   = 16;
  static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
  static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
  static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

  /** Returns the number of shared holds represented in count  */
  static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
  /** Returns the number of exclusive holds represented in count  */
  static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```
用state 的高 16 位代表读锁的获取次数；用state 的低 16 位代表写锁的获取次数
如图所示：（网络盗图）

![upload successful](/images/ReentrantReadWriteLock__0.png)

分别用16位来保存读和写的状态，即同时读的线程数和同时写的线程数（重入的次数，因为写锁是独占模式）。通过位运算来获得高16位或低16位的数。

在看看读锁和写锁在ReentrantReadWriteLock内部的表示方式
```java
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    //默认非公平的策略 
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```
可以看出内部分别持有一个`ReadLock`和`WriteLock`对象。从api得知上锁和解锁的操作都是由这两类完成的，下面将详细分析这两个类的作用。

### 写锁WriteLock
`WriteLock`是ReentrantReadWriteLock的内部类，它持有一个`Sync`的实例。如下代码：

```java
        private final Sync sync;
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

看看有哪些方法：
![upload successful](/images/ReentrantReadWriteLock__1.png)

跟ReentrantLock长得很像，用法都类似，有lock，tryLock，支持中断方式的，支持超时的，释放锁等。
由于`Sync`继承`AbstractQueuedSynchronizer`，所以Sync的用法其实都是在AQS这个框架里的，在下面源码分析中，不会过多解释AQS部分的代码，希望读者提前了解AQS。

着重分析`lock`、`tryLock`、`unlock`方法，其他方法也是差不多的。

#### lock
写锁说明：
- 写锁是独占锁。
- 如果有读锁被占用，写锁获取是要进入到阻塞队列中等待的，即读写互斥。

```java
        public void lock() {
            sync.acquire(1);
        }
```
调用Sync的`acquire`，这是AQS的方法，最终还是会调用`tryAcquire`这个方法
```java
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            //获取写锁数
            int w = exclusiveCount(c);
            //c != 0 说明存在读锁或写锁或两者都有
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //如上解释，如果c != 0 and w == 0，那么存在读锁。由于锁不能升级（即自身在获取读锁的情况下，再获取写锁，反之则可以，下文会说）
                //另一种情况是，存在读锁或写锁，只要持锁者不是自己，那么获取写锁失败，返回false。
                //这里返回的false，跟AQS有关，AQS会把此线程放进CLH阻塞队列
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //以上情况都不是，那么就是重入的情况，直接AQS的setState（由于写锁标记是用低16位，所以直接相加即可）
                setState(c + acquires);
                return true;
            }
            //到这里说明 c == 0，即锁此刻没有被占用，但是可能有人在排队（排队的哥们可能来不及获得锁）
            //writerShouldBlock顾名思义即“是否应该阻塞”，“阻塞”不好理解，我觉得用“竞争”会好点。现在锁没被占用，“我”是否应该去“占用”它呢？这个跟公平策略有关（下文会说），如果是公平的话，那么会判断现在是否有人在排队，有的话，那么不好意思，返回false；不公平的话，那么可以竞争cas一下，看看运气。
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            //标记锁持有线程为当前线程，用于可重入判断。
            setExclusiveOwnerThread(current);
            return true;
        }
```


#### tryLock
`tryLock`直接调用`Sync`的`tryWriteLock`方法，它跟`lock`不同，它不会阻塞，获取锁失败直接返回false，所以这个方法是不经过AQS的。
```java
       public boolean tryLock( ) {
            return sync.tryWriteLock();
        }
```

```java
        /**
         * Performs tryLock for write, enabling barging in both modes.
         * This is identical in effect to tryAcquire except for lack
         * of calls to writerShouldBlock.
         */
        final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                //根据lock方法一样判断（解释看上面）
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            //也是跟lock方法一样的操作
            //注意，这里没有了公平/不公平的说了，跟ReentrantLock一样，tryLock不管3721直接竞争，也可以理解为是不公平的。
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

#### unlock
```java
        public void unlock() {
            sync.release(1);
        }
```

```java
        /*
         * Note that tryRelease and tryAcquire can be called by
         * Conditions. So it is possible that their arguments contain
         * both read and write holds that are all released during a
         * condition wait and re-established in tryAcquire.
         */
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            //这里操作很简单，因为写锁是独占的，所以线程安全
            setState(nextc);
            return free;
        }
```

### 读锁ReadLock
`ReadLock`是ReentrantReadWriteLock的内部类，它持有一个`Sync`的实例。如下代码：
```java
        private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```
看看有哪些方法：


![upload successful](/images/ReentrantReadWriteLock__3.png)

跟`WriteLock`长得很像，也是重分析`lock`、`tryLock`、`unlock`方法。

#### 读锁线程缓存
ReentrantReadWriteLock对读锁的线程及读锁次数有缓存，来增加性能，因为在下文会提到，所以先了解一下比较好
```java
        /**
         * A counter for per-thread read hold counts.
         * Maintained as a ThreadLocal; cached in cachedHoldCounter
         */
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }

        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         */
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }

        //组合使用上面两个类，用一个 ThreadLocal 来记录当前线程持有的读锁数量
        private transient ThreadLocalHoldCounter readHolds;

        //缓存最后一个获取读锁的线程
        private transient HoldCounter cachedHoldCounter;

        // 第一个获取读锁的线程(并且其未释放读锁)，以及它持有的读锁数量
        private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;

        Sync() {
            readHolds = new ThreadLocalHoldCounter();
            setState(getState()); // ensures visibility of readHolds
        }
```

#### lock
读锁的获取，由于读锁是共享的，所以用到了AQS共享模式那套api。
一样的套路，`lock`方法调用的是AQS的`acquireShared`方法，而最终会先调用`Sync`的`tryAcquireShared`方法

```java
        public void lock() {
            sync.acquireShared(1);
        }

        public final void acquireShared(int arg) {
            if (tryAcquireShared(arg) < 0)
                doAcquireShared(arg);
        }
```

```java
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            //exclusiveCount(c) != 0 说明有写锁，如果写锁不是自己的，则失败
            //如果写锁自己的，那么是可以的，这叫锁降级，即在自身获取写锁的情况下允许获取读锁，反之则不行
            //失败则返回-1，结合AQS来看，如果tryAcquireShared返回小于0，那么就要乖乖进阻塞队列
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            //读锁的次数
            int r = sharedCount(c);
            //注意，来到这步，说明是 exclusiveCount(c) == 0 或 getExclusiveOwnerThread() == current，即没有写锁或当前持锁者是自己，这两种情况都是可以获得读锁的。
            //readerShouldBlock“读锁是否应该阻塞”，这跟是否公平策略有关（下文会讲），公平策略下看看有没有人在排队；非公平策略下看有没有写请求在队列头
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //允许获得读锁且cas成功
                if (r == 0) {
                    //r == 0 说明是第一个获取读锁的线程，那么记录firstReader和firstReaderHoldCount
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //如果第一个获取读锁的线程再次获得读锁，只需firstReaderHoldCount++
                    firstReaderHoldCount++;
                } else {
                    //到这里，当前线程不是第一个获得读锁的线程，而且前面已经有其他线程获得了读锁
                    //cachedHoldCounter是用于缓存最后一个获取读锁的线程，这里作用是记录当前线程及读锁次数到threadlocal变量中
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    //读锁次数++
                    rh.count++;
                }
                //返回大于0给AQS，说明获取锁成功
                return 1;
            }
            //来到这步，总结一下现在的情况：
            //在没有写锁或当前持锁者是自己的情况下，有两个分支：
            //1、readerShouldBlock返回true，这个受公平策略影响，反正就是队列前面已经有人排队了（写或读请求），不给我插队
            //2、我可以插队，但是ca失败了
            return fullTryAcquireShared(current);
        }

        /**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        //主要用来处理cas失败和重入的情况
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            //官方说了，这段代码有点”重复多余“
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    //根据tryAcquireShared一样，有写锁且写锁不是自己的，则失败
                    //这个地方是重复检查，因为在tryAcquireShared已经检查过一次了，但在高并发下，可能已经有写锁产生了。如果还是这样，那么就死心了。
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    //到这里说明:
                    //1、exclusiveCount(c) == 0 即没有写锁
                    //2、readerShouldBlock返回true，即有请求在排队
                    if (firstReader == current) {
                        //？？这里有点不明白
                    } else {
                        //readHolder第一次会做初始化
                        if (rh == null) {
                            //rh等于上一次获得读锁的线程
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                //如果为null或不等于当前线程，则从threadlocal中取
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        //如果当前线程都没有获取过读锁的，那么返回失败，乖乖排队去吧。（如果获取过，那么当重入处理）
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //到这里，是允许加读锁或者是可重入的情况，那么就cas一下
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    //处理一下读线程相关的缓存变量
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

```
总结一下：
`tryAcquireShared`整体代码算有点多，也算有点复杂，不过多看几次就会发现代码有一定的重复性，看代码时要想清楚有哪些情景，在这里你会发现情景无非就几种，加上对读锁线程的缓存和threadlocal的处理可能会对你来说增加了一定的复杂度，不过多看几次就好了。

#### tryLock
`tryLock`不过AQS，整体处理跟`tryAcquireShared`类似，而且更加简单，因为不用考虑CLH队列的情况。
```java
        public boolean tryLock() {
            return sync.tryReadLock();
        }
```

```java
        /**
         * Performs tryLock for read, enabling barging in both modes.
         * This is identical in effect to tryAcquireShared except for
         * lack of calls to readerShouldBlock.
         */
        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                //跟tryAcquireShared一样，熟悉的判断
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //cas一下，也是跟tryAcquireShared的处理一样
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```
在熟悉`tryAcquireShared`后，你会发现`tryReadLock`不难，而且处理流程几乎一致。

#### unlock
AQS的套路`unlock` -> `releaseShared` -> `tryReleaseShared`。
```java
        public void unlock() {
            sync.releaseShared(1);
        }
        
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            //对读锁缓存的处理
            if (firstReader == current) {
                //如果是第一个获得读锁的线程释放锁，直接firstReader = null
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                //或者读锁次数减一
                    firstReaderHoldCount--;
            } else {
                //处理cachedHoldCounter和ThreadLocal
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            //处理state变量的值
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

### 特性
ReentrantReadWriteLock支持一些特性
- 支持公平/非公平
- 锁降级
- 锁重入
- 可中断
- 支持Condition条件

我们着重看看前面两个特性，后面三个是跟ReentrantLock一样的。

#### 公平/非公平
公平策略在ReentrantLock也有，不过在ReentrantReadWriteLock中对于读锁和写锁的处理是有点不同的的。
其实在上面加锁过程的分析中也提到了对于公平和非公平的处理，下面再详细看看。

公平锁由`FairSync`实现，继承`Sync`；非公平锁由`NonfairSync`实现，继承`Sync`。
它们两者都是重写了`writerShouldBlock`和`readerShouldBlock`方法，对于不同策略下的读写是否应该阻塞的判断。

```java
    /**
     * Nonfair version of Sync
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        //非公平锁下的写，永远不应该被阻塞，写是更加优先
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        //非公平锁下的读，要看看有没有写请求在排队，有就阻塞，因为读写互斥
        //这里只判断了队列头，因为是非公平嘛
        //apparentlyFirstQueuedIsExclusive是AQS提供的方法
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }

    /**
     * Fair version of Sync
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        //公平锁下的写和读都要看有没有人在排队，因为是公平嘛。
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

#### 锁降级
锁降级上面也提到过，意思是在自身获得写锁的情况下，可以无条件获得读锁，反之则不行。
看看官方的描述
> Lock downgrading
Reentrancy also allows downgrading from the write lock to a read lock,
by acquiring the write lock, then the read lock and then releasing the
write lock. However, upgrading from a read lock to the write lock is not possible.

这个缓存数据读写的官方demo，对于锁降级还是读写锁的使用都有参考的价值
```java
  class CachedData {
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
    void processCachedData() {
      rwl.readLock().lock();
      if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
      }
 
      try {
        use(data);
      } finally {
        rwl.readLock().unlock();
      }
    }
  }
```

## 总结
ReentrantReadWriteLock在读多写少的场景下，有更高的性能；
ReentrantReadWriteLock是基于AbstractQueuedSynchronizer实现的，利用AQS的state状态的高16位表示读状态，低16位表示写状态，达到读写互斥的效果；支持公平/非公平的策略，锁降级等特性。
最后再补充一下，ReentrantReadWriteLock的写表示独占锁，读表示共享锁，所以ReentrantReadWriteLock是用到了AQS的两种模式，在CLH阻塞队列中有可能同时存在共享节点和独占节点。

如图所示：

![overwrote existing file](/images/ReentrantReadWriteLock__2.png)


## 参考资料
https://juejin.im/post/5b7a834551882542c20f1985
https://javadoop.com/post/reentrant-read-write-lock