title: ThreadLocal的使用及原理
author: alben.wong
abbrlink: d7ec7cff
tags:
  - threadlocal
categories:
  - java
keywords: threadlocal java
description: ThreadLocal线程级别变量的使用及其原理分析，如果你还不知道threadlocal，那你就要了解一下，相信你一定会用到它。
date: 2019-02-12 09:45:00
---
## 概要
如果你还不知道threadlocal，那你就要了解一下，相信你一定会用到它。

## 作用
threadlocal最大作用就是提供线程级别的变量生命周期。
试想，如果你需要一个变量在一个线程的生命周期内都可以访问到，在不使用threadlocal的前提下你会怎么做？你或许这样做  

1. 提供一个类级别或者静态变量  
但是这个方法大家很容易就想到在高并发时会出问题。

2. 把这个局部变量一直传递下去  
但是如果你要调用的方法层次很深呢？难道你对每个方法都增加一个参数吗？显然不实际。

所以threadlocal就是提供了一个可行的方案，使得这个变量可以随时访问到，并且不会跟其他线程产生冲突。


## 使用

threadlocal的使用很简单，就是一个get, set。

```java
public class ThreadLocalTest {

    //定义一个ThreadLocal的变量, 需要指定类型
    public static ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();

    @Before
    public void init(){
    	#set值进去
        threadLocal.set("test");
    }

    @Test
    public void test() {
    	//在需要时get出来
        System.out.println("threadLocal's value=" + threadLocal.get());
    }
    
}
```


## 实现原理

### set
我们先从set方法入手看看做了手脚。

```java
	public void set(T value) {
    	//取出当前线程
        Thread t = Thread.currentThread();
        //根据当前线程获取ThreadLocalMap。从getMap方法可以看到这个ThreadLocalMap就是保存在Thread对象里面
        ThreadLocalMap map = getMap(t);
        if (map != null）
        	#map已经存在就是set进去
            map.set(this, value);
        else
        	#不存在新建一个map
            createMap(t, value);
    }
    
    //getMap方法
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    //createMap方法
    //注意，这里的this指的是ThreadLocal对象
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
     
```

我们再看看ThreadLocalMap的创建及其他方法。ThreadLocalMap是定义在ThreadLocal里的一个静态类。

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            //通过ThreadLocal的hashCode确定index，这个我们稍后再说
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            //重点，把ThreadLocal对象作为key存到了ThreadLocalMap的Entry里
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```

set方法比较简单，跟普通的Map差不多，把key和value set进去。
里面还包含了清理key为null的Entry对象的一些操作。

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

### get

```java
public T get() {
		//取当前线程
        Thread t = Thread.currentThread();
        //取出线程的ThreadLocalMap，跟上面是一样的
        ThreadLocalMap map = getMap(t);
        if (map != null) {
        	//根据当前对象ThreadLocal取出ThreadLocalMap.Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //如果map为null则做初始化，跟上面的createMap差不多
        return setInitialValue();
    }
```

### 图解关系

![upload successful](/images/ThreadLocal的使用及原理_0.png)

可以看出相同的ThreadLocal在不同的线程有不同的值。

主要记住ThreadLocal是作为ThreadLocalMap的key，可能开始有点绕，但是慢慢思考，理清它们的关系就行了。


## 两个问题
### 内存泄漏 ？
这是一个对ThreadLocal来说老生常谈的问题了。那使用ThreadLocal为什么会导致内存泄漏？还有我们应该怎么去避免？是我们应该关注的两个点。

#### 原因
首先，我这里假设大家对java的内存回收机制和引用（Reference）有一定的了解。如果不知道，请自行google了。

我们先看看ThreadLocalMap的Entry的定义
```java
//对key使用了WeakReference
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

为什么要使用WeakReference？  
网上很多的说法都说是使用了弱引用就会被GC一句带过，我觉得很多都说得不清不楚的。我通过自己的理解向大家解析一下：  
其实很简单，从变量的作用域及引用关系的角度出发思考。试想如果一个ThreadLocal定义为一个类实例的变量或者是一个方法内的局部变量，那么当这个类实例被销毁了或方法退出了，在理想的情况下，垃圾回收器应该回收掉这个ThreadLocal是吧，毕竟它的生命周期已经完结了，但是如果这时ThreadLocalMap还是持有这个ThreadLocal的强引用的话，这个ThreadLocal就不会被回收，直到这个ThreadLocalMap被销毁或者这个线程被销毁。  
说白了，从上面那个图看出，这样的设计导致的结果是这个ThreadLocal的生命周期跟线程的生命周期挂上钩了。


同时，这里又会出现另外一种内存泄漏的问题，即使ThreadLocal回收了，但是value没有被回收，还是会导致内存泄漏。但是你没办法把value设置为WeakReference，因为value不是你的，不归你管。


#### 如何防止
ThreadLocal采用如下解决内存泄漏，看expungeStaleEntry方法。
```java
/**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

在ThreadLocal的源码你会看到很多清洗数据的代码最终都会调用到这个方法。这个方法主要逻辑，简单来说就是当key为null，把value也设置为null，从而让value也被回收。
这个方法的触发点有很多，当对ThreadLocal进行set，get，remove等操作时都会。


容器(如tomcat，netty)一般都是使用线程池处理用户到请求，此时用ThreadLocal要特别注意内存泄漏的问题，一个请求结束了，处理它的线程也结束，但此时这个线程并没有死掉，它只是归还到了线程池中，这时候应该清理掉属于它的ThreadLocal信息。

所以我们使用ThreadLocal一个比较好的习惯是在finally块调用remove方法。


### hashcode和0x61c88647？

既然ThreadLocal用map就避免不了冲突的产生。

在ThreadLocalMap的构造方法中，我们可以看到以下代码

```
	//table的下标的计算方式
	int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
	table[i] = new Entry(firstKey, firstValue);
```


```
	//threadLocalHashCode的定义
	private final int threadLocalHashCode = nextHashCode();

	private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    //HASH_INCREMENT的十进制为1640531527
    private static final int HASH_INCREMENT = 0x61c88647;
    
```

从代码可以看出每个ThreadLocal的实例的threadLocalHashCode的差值为0x61c88647这么多，那为什么要这样做呢？

这个魔数的选取与斐波那契散列有关，0x61c88647对应的十进制为1640531527。  
斐波那契散列的乘数可以用(long) ((1L << 31) \* (Math.sqrt(5) - 1))可以得到2654435769，如果把这个值给转为带符号的int，则会得到-1640531527。换句话说
(1L << 32) - (long) ((1L << 31) \* (Math.sqrt(5) - 1))得到的结果就是1640531527也就是0x61c88647。  
通过理论与实践，当我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。  
ThreadLocalMap使用的是线性探测法，均匀分布的好处在于很快就能探测到下一个临近的可用slot，从而保证效率。。为了优化效率。

简单来说就是在table\[\]的size为2的次幂情况下，取模会得到均匀分布。



## 一个优化点
从上面得知，ThreadLocal的Map可能会产生冲突，解决冲突的办法是线性探测。  
而Netty的FastThreadLocal的利用了一个自增序号来作为下标，避免了冲突的产生。  
```
public FastThreadLocal() {
        index = InternalThreadLocalMap.nextVariableIndex();
    }
    
    public static int nextVariableIndex() {
        int index = nextIndex.getAndIncrement();
        if (index < 0) {
            nextIndex.decrementAndGet();
            throw new IllegalStateException("too many thread-local indexed variables");
        }
        return index;
    }
```


## 参考资料
https://juejin.im/post/5b5ecf9de51d45190a434308