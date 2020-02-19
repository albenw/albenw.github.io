title: MpscGrowableArrayQueue分析
author: alben.wong
abbrlink: f00ab7dc
tags:
  - mpsc
  - jctools
categories:
  - java
date: 2020-02-17 10:02:00
keywords: mpsc jctools
description: MpscGrowableArrayQueue是JCTools里的一个工具，是对于特定场景化的定制，即MPSC（Multi-Producer & Single-Consumer），在这种场景下，相对于BlockingQueue，能够满足高性能的需要。
---
## 概要

MpscGrowableArrayQueue是JCTools里的一个工具，是对于特定场景化的定制，即MPSC（Multi-Producer & Single-Consumer），在这种场景下，相对于BlockingQueue，能够满足高性能的需要。



## 背景

JCTools是一款对jdk并发数据结构进行增强的并发工具，主要提供了map以及queue的增强数据结构。

Mpsc**ArrayQueue是由JCTools提供的一个多生产者单个消费者的数组队列。多个生产者同时并发的访问队列是线程安全的，但是同一时刻只允许一个消费者访问队列，这是需要程序控制的，因为MpscQueue的用途即为多个生成者可同时访问队列，但只有一个消费者会访问队列的情况。如果是其他情况你可以使用JCTools提供的其他队列。



## 应用场景

上面说了MpscGrowableArrayQueue是用于特定化场景，即MPSC。其实这种场景我们平时也会看到很多，在各种框架、工具中也有它的身影。

### Netty

原来Netty还是自己写的MpscLinkedQueueNode，后来新版本就换成使用JCTools的并发队列了。

Netty的线程模型决定了taskQueue可以用多个生产者线程同时提交任务，但只会有EventLoop所在线程来消费taskQueue队列中的任务。这样JCTools提供的MpscQueue完全符合Netty线程模式的使用场景。而LinkedBlockingQueue会在生产者线程操作队列时以及消费者线程操作队列时都对队列加锁以保证线程安全性。虽然，在Netty的线程模型中程序会控制访问taskQueue的始终都会是EventLoop所在线程，这时会使用偏向锁来降低线程获得锁的代价。



### Caffeine

像我的一篇文章说的（{% post_link Caffeine高性能设计剖析  %}），如果对Caffeine设置了expireAfterWrite或refreshAfterWrite，那么每次写操作都会把afterWrite的task放在一个MpscGrowableArrayQueue里，之后再异步处理这些task。一般写操作有可能并发进行，有多个生产者， 但是只用一个线程来处理，来降低复杂度，这里的场景就很适合mpsc了。



## 使用

MpscGrowableArrayQueue的使用跟其他queue类似，提供`offer`, `poll`, `peek`等常规方法，但由于特定化的场景，由于设计上的原因，做了一点限制，相当于牺牲了一些功能，不支持这三个方法：remove(Object o)，removeAll(Collection)，retainAll(Collection)。



## 原理分析

BlockingQueue对于每次的读写都会使用锁Lock来阻塞操作，这样在高并发下会产生性能问题，影响程序的吞吐量，那么对于这种情况的优化，很自然就会想到要把锁去掉，采用Lock-free的设计，这是生产端的原理；对于消费端，干脆只限制只有一个线程来使用（没有强制限制），那么就不存在并发问题了。

下面我们来看看MpscGrowableArrayQueue的具体实现（源码基于jctools的3.0.0版本

）：



###基本属性

先看看MpscGrowableArrayQueue的继承关系

![upload successful](/images/MpscGrowableArrayQueue分析__0.png)

重点看红框框着的，其他都是用于padding的。`MpscGrowableArrayQueue`和`MpscChunkedArrayQueue`的功能差不多，他们都继承于`BaseMpscLinkedArrayQueue`类。

相对于其他Mpsc**Queue类，MpscChunkedArrayQueue根据名字可以看出它是基于数组实现，跟准确的说是数组链表。这点可从它的父类BaseMpscLinkedArrayQueue看出。它融合了链表和数组，既可以动态变化长度，同时不会像链表频繁分配Node。并且吞吐量优于传统的链表。

BaseMpscLinkedArrayQueue实现了绝大部分的核心功能，下面讲到的源码都在这个类里面。



看看官方对MpscGrowableArrayQueue的定义：

```java
/**
 * An MPSC array queue which starts at <i>initialCapacity</i> and grows to <i>maxCapacity</i> in linked chunks,
 * doubling theirs size every time until the full blown backing array is used.
 * The queue grows only when the current chunk is full and elements are not copied on
 * resize, instead a link to the new chunk is stored in the old chunk for the consumer to follow.
 */
public class MpscGrowableArrayQueue<E> extends MpscChunkedArrayQueue<E>
```

它是一个可自动扩展容量的array，扩展时它不会像hashmap那样把旧数组的元素复制到新的数组，而是用一个link来连接到新的数组。



`BaseMpscLinkedArrayQueue`的主要几个属性，这几个属性比较分散（由于用了很多的padding类来做继承），我把他们集中起来：

```java
    //生产者的最大下标
    private volatile long producerLimit;
    //生产者数组的mask，偏移量offset & mask得到在数组的位置
    protected long producerMask;
    //生产者的buffer，由于会产生的新的数组buffer，所以生产者和消费者各自维护自己的buffer
    protected E[] producerBuffer;
    //生产者的当前下标
    private volatile long producerIndex;
    
    //消费者的当前下标
    private volatile long consumerIndex;
    //消费者数组的mask
    protected long consumerMask;
    //生产者的buffer
    protected E[] consumerBuffer;
```



### 初始化

```java
    /**
     * @param initialCapacity the queue initial capacity. If chunk size is fixed this will be the chunk size.
     *                        Must be 2 or more.
     * @param maxCapacity     the maximum capacity will be rounded up to the closest power of 2 and will be the
     *                        upper limit of number of elements in this queue. Must be 4 or more and round up to a larger
     *                        power of 2 than initialCapacity.
     */
    // 可以指定init size，或者不指定
    public MpscChunkedArrayQueue(int initialCapacity, int maxCapacity){
        super(initialCapacity, maxCapacity);
    }
    
    public MpscChunkedArrayQueue(int maxCapacity){
        super(max(2, min(1024, roundToPowerOfTwo(maxCapacity / 8))), maxCapacity);
    }
    
    MpscChunkedArrayQueueColdProducerFields(int initialCapacity, int maxCapacity){
        super(initialCapacity);
        RangeUtil.checkGreaterThanOrEqual(maxCapacity, 4, "maxCapacity");
        RangeUtil.checkLessThan(roundToPowerOfTwo(initialCapacity), roundToPowerOfTwo(maxCapacity),
            "initialCapacity");
        //数组有一个最大的限制，最大值为Pow2(max)*2，例如你指定max=100，maxQueueCapacity=256
        maxQueueCapacity = ((long) Pow2.roundToPowerOfTwo(maxCapacity)) << 1;
    }

    public BaseMpscLinkedArrayQueue(final int initialCapacity){
        RangeUtil.checkGreaterThanOrEqual(initialCapacity, 2, "initialCapacity");
        //初始化的size为2的整数倍
        int p2capacity = Pow2.roundToPowerOfTwo(initialCapacity);
        //这里mask为什么不是p2capacity - 1，是因为把最后一位当作是否resizing的标记
        //所以这里的容量是虚扩成2倍，所以后面看到下标index追加时是加2，而不是1
        //而且在获取数组的偏移量offset时，把数组的arrayIndexScale也除以了2
        //所以要注意index和limit的数值都为实际容量的2倍
        long mask = (p2capacity - 1) << 1;
        //初始化数组
        E[] buffer = allocateRefArray(p2capacity + 1);
        //生产者的数组指向初始化数组
        producerBuffer = buffer;
        //buffer对应的mask
        producerMask = mask;
        //消费者的数组指向初始化数组
        consumerBuffer = buffer;
        //（同上）
        consumerMask = mask;
        //生产者的最大值limit等于mask
        soProducerLimit(mask);
    }
```

可以看出初始化时初始化了init size的数组，以及mask，limit等。

注意，producer和consumer的index初始值都为0，这里没写。



### offer

一个queue需要往里面加入元素，offer操作是其最基本的功能。

```java
    public boolean offer(final E e) {
        if (null == e)
        {
            throw new NullPointerException();
        }

        long mask;
        E[] buffer;
        long pIndex;

        while (true)
        {
            long producerLimit = lvProducerLimit();
            pIndex = lvProducerIndex();
            //最低位为1说明在resize，继续自转等resize结束
            if ((pIndex & 1) == 1)
            {
                continue;
            }
            
            mask = this.producerMask;
            buffer = this.producerBuffer;
            
            //pIndex >= producerLimit有几种可能
            //idnex小于等于limit，说明生产者的位置达到了数组的“末端”，这里用引号是因为实际上不是，准确来说是可允许存放元素的位置
            if (producerLimit <= pIndex)
            {
                //通过offerSlowPath判断该进行什么操作，retry还是resize等（下面展述）
                int result = offerSlowPath(mask, pIndex, producerLimit);
                switch (result)
                {
                    case CONTINUE_TO_P_INDEX_CAS:
                        break;
                    case RETRY:
                        continue;
                    case QUEUE_FULL:
                        return false;
                    case QUEUE_RESIZE:
                        //扩容（下面展述）
                        resize(mask, buffer, pIndex, e, null);
                        return true;
                }
            }
            //走到这里说明producerLimit > pIndex，即允许添加元素
            //用当前的index进行cas操作来设置值，成功则退出；失败说明存在并发，则继续while循环
            if (casProducerIndex(pIndex, pIndex + 2))
            {
                break;
            }
        }
        //这里通过arrayBaseOffset和arrayIndexScale获取数组的偏移量
        final long offset = modifiedCalcCircularRefElementOffset(pIndex, mask);
        //根据偏移量offset把元素设进去
        soRefElement(buffer, offset, e);
        return true;
    }
```



offerSlowPath方法：

```java
    
    private int offerSlowPath(long mask, long pIndex, long producerLimit)
    {
        //消费者下标 
        final long cIndex = lvConsumerIndex();
        //用当前mask获取当前buffer的size（其实就是mask）
        long bufferCapacity = getCurrentBufferCapacity(mask);
        //如果已经有消费的话，那么下面条件会成立
        //如果是这种情况，就需要把已经消费的空位利用起来，像ring buffer一样
        if (cIndex + bufferCapacity > pIndex)
        {
            //通过cas设置limit，limit的大小为
            if (!casProducerLimit(producerLimit, cIndex + bufferCapacity))
            {
                //失败则重试
                return RETRY;
            }
            else
            {
                //成功则出去cas pIndex
                return CONTINUE_TO_P_INDEX_CAS;
            }
        }
        //检查是否有空位，因为queue在初始化时有设置maxQueueCapacity
        else if (availableInQueue(pIndex, cIndex) <= 0)
        {
            //返回full
            return QUEUE_FULL;
        }
        //来到这里说要扩容了，把最后index的最后以为置为1，表示resizing
        else if (casProducerIndex(pIndex, pIndex + 1))
        {
            //返回resize
            return QUEUE_RESIZE;
        }
        else
        {
            //说明已经resize了，返回继续重试，等待resize完成
            return RETRY;
        }
    }
```



resize方法：

```java
    private void resize(long oldMask, E[] oldBuffer, long pIndex, E e, Supplier<E> s)
    {
        assert (e != null && s == null) || (e == null || s != null);
        //newBuffer的大小为原来的2倍
        int newBufferLength = getNextBufferSize(oldBuffer);
        final E[] newBuffer;
        try
        {   
            //新建newBuffer数组
            newBuffer = allocateRefArray(newBufferLength);
        }
        catch (OutOfMemoryError oom)
        {
            assert lvProducerIndex() == pIndex + 1;
            soProducerIndex(pIndex);
            throw oom;
        }
        //生产者的buffer指向newBuffer
        producerBuffer = newBuffer;
        //新的mask
        final int newMask = (newBufferLength - 2) << 1;
        producerMask = newMask;

        //计算pIndex在旧数组的偏移
        final long offsetInOld = modifiedCalcCircularRefElementOffset(pIndex, oldMask);
        //计算pIndex在新数组的偏移
        final long offsetInNew = modifiedCalcCircularRefElementOffset(pIndex, newMask);
        //设置新加入的元素到新数组
        soRefElement(newBuffer, offsetInNew, e == null ? s.get() : e);
        //旧数组的最后一个位置指向新的数组
        soRefElement(oldBuffer, nextArrayOffset(oldMask), newBuffer);

        final long cIndex = lvConsumerIndex();
        final long availableInQueue = availableInQueue(pIndex, cIndex);
        RangeUtil.checkPositive(availableInQueue, "availableInQueue");

        //更新limit
        soProducerLimit(pIndex + Math.min(newMask, availableInQueue));

        //更新pIndex
        soProducerIndex(pIndex + 2);

        //pIndex在旧数组的位置设置一个固定值-JUMP，来告诉要跳到下一个数组
        soRefElement(oldBuffer, offsetInOld, JUMP);
    }
```



### poll

poll方法：

注意这个方法会有并发问题，所以MPSC的使用要求是一次只用一个线程来poll。

```java
    public E poll()
    {
        final E[] buffer = consumerBuffer;
        final long index = lpConsumerIndex();
        final long mask = consumerMask;

        //获取数组的偏移量
        final long offset = modifiedCalcCircularRefElementOffset(index, mask);
        //获取数据
        Object e = lvRefElement(buffer, offset);
        if (e == null)
        {
            if (index != lvProducerIndex())
            {
                //正常来说不会为null，但是在offer添加数据时使用putOrderedObject，即lazySet，在并发时可能会发生，所以while继续读取
                do
                {
                    e = lvRefElement(buffer, offset);
                }
                while (e == null);
            }
            //cIndex == pIndex 说明没有数据
            else
            {
                return null;
            }
        }

        //link到下一个数组
        if (e == JUMP)
        {
            final E[] nextBuffer = nextBuffer(buffer, mask);
            return newBufferPoll(nextBuffer, index);
        }

        //置为null来释放内存
        soRefElement(buffer, offset, null);
        soConsumerIndex(index + 2);
        return (E) e;
    }
```



### 结构图

![upload successful](/images/MpscGrowableArrayQueue分析__1.png)




## 总结

在多生产者单消费者的场景下，使用`MpscGrowableArrayQueue`可以满足高性能的队列读写要求。

MpscGrowableArrayQueue不再使用Lock来阻塞操作，而是使用CAS来操作，包括使用putOrderedObject来进行快速set、使用arrayBaseOffset和arrayIndexScale来计算数组的偏移量等等；

还有padding来解决伪共享问题。



## 参考资料

[Which Queue Should I Use?](https://github.com/JCTools/JCTools/wiki/Which-Queue-Should-I-Use%3F)
