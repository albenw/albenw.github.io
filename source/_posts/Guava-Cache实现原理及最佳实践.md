title: Guava Cache实现原理及最佳实践
author: alben.wong
abbrlink: df42dc84
tags:
  - guava-cache
categories:
  - java
  - guava
keywords: guava cache 源码 原理 最佳实践
description: 本文内容包括Guava Cache的使用、核心机制的讲解、核心源代码的分析以及最佳实践的说明。
date: 2019-07-28 20:42:00
---
## 概要

Guava Cache是一款非常优秀本地缓存，使用起来非常灵活，功能也十分强大。Guava Cache说简单点就是一个支持LRU的ConcurrentHashMap，并提供了基于容量，时间和引用的缓存回收方式。

本文详细的介绍了Guava Cache的使用注意事项，即最佳实践，以及作为一个Local Cache的实现原理。



## 应用及使用

### 应用场景

- 读取热点数据，以空间换时间，提升时效
- 计数器，例如可以利用基于时间的过期机制作为限流计数

### 基本使用

Guava Cache提供了非常友好的基于Builder构建者模式的构造器，用户只需要根据需求设置好各种参数即可使用。Guava Cache提供了两种方式创建一个Cache。

#### CacheLoader

CacheLoader可以理解为一个固定的加载器，在创建Cache时指定，然后简单地重写V load(K key) throws Exception方法，就可以达到当检索不存在的时候，会自动的加载数据的。例子代码如下：

```java
//创建一个LoadingCache，并可以进行一些简单的缓存配置
private static LoadingCache<String, String > loadingCache = CacheBuilder.newBuilder()
            //最大容量为100（基于容量进行回收）
            .maximumSize(100)
            //配置写入后多久使缓存过期-下文会讲述
            .expireAfterWrite(150, TimeUnit.SECONDS)
            //配置写入后多久刷新缓存-下文会讲述
            .refreshAfterWrite(1, TimeUnit.SECONDS)
            //key使用弱引用-WeakReference
            .weakKeys()
            //当Entry被移除时的监听器
            .removalListener(notification -> log.info("notification={}", GsonUtil.toJson(notification)))
            //创建一个CacheLoader，重写load方法，以实现"当get时缓存不存在，则load，放到缓存，并返回"的效果
            .build(new CacheLoader<String, String>() {
              	//重点，自动写缓存数据的方法，必须要实现
                @Override
                public String load(String key) throws Exception {
                    return "value_" + key;
                }
              	//异步刷新缓存-下文会讲述
                @Override
                public ListenableFuture<String> reload(String key, String oldValue) throws Exception {
                    return super.reload(key, oldValue);
                }
            });

    @Test
    public void getTest() throws Exception {
      	//测试例子，调用其get方法，cache会自动加载并返回
        String value = loadingCache.get("1");
      	//返回value_1
        log.info("value={}", value);
    }

```



#### Callable

在上面的build方法中是可以不用创建CacheLoader的，不管有没有CacheLoader，都是支持Callable的。Callable在get时可以指定，效果跟CacheLoader一样，区别就是两者定义的时间点不一样，Callable更加灵活，可以理解为Callable是对CacheLoader的扩展。例子代码如下：

```java
    @Test
    public void callableTest() throws Exception {
        String key = "1";
        //loadingCache的定义跟上一面一样
        //get时定义一个Callable
        String value = loadingCache.get(key, new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "call_" + key;
            }
        });
        log.info("call value={}", value);
    }
```



#### 其他用法

显式插入：

支持loadingCache.put(key, value)方法直接覆盖key的值。

显式失效：

支持loadingCache.invalidate(key) 或 loadingCache.invalidateAll() 方法，手动使缓存失效。



## 缓存失效机制

Guava Cache有一套十分优秀的缓存失效机制，这里主要介绍的是基于时间的失效回收。

缓存失效的目的是让缓存进行重新加载，即刷新，使调用者可以正常访问获取到最新的数据，而不至于返回null或者直接访问DB。

从上面的例子中我们知道与失效/缓存刷新相关配置有 expireAfterWrite / expireAfterAccess、refreshAfterWrite 还有 CacheLoader的reload方法。



### 一般用法

#### expireAfterWrite/expireAfterAccess

##### 使用背景

如果对缓存设置过期时间，在高并发下同时执行get操作，而此时缓存值已过期了，如果没有保护措施，则会导致大量线程同时调用生成缓存值的方法，比如从数据库读取，对数据库造成压力，这也就是我们常说的“缓存击穿”。

##### 做法

而Guava cache则对此种情况有一定控制。当大量线程用相同的key获取缓存值时，只会有一个线程进入load方法，而其他线程则等待，直到缓存值被生成。这样也就避免了缓存击穿的危险。这两个配置的区别前者记录写入时间，后者记录写入或访问时间，内部分别用writeQueue和accessQueue维护。

PS: 但是在高并发下，这样还是会阻塞大量线程。



#### refreshAfterWrite

##### 使用背景

使用 expireAfterWrite 会导致其他线程阻塞。

##### 做法

更新线程调用load方法更新该缓存，其他请求线程返回该缓存的旧值。



#### **异步刷新** 

##### 使用背景

单个key并发下，使用refreshAfterWrite，虽然不会阻塞了，但是如果恰巧同时多个key同时过期，还是会给数据库造成压力，这就是我们所说的“**缓存雪崩**”。

##### 做法

这时就要用到异步刷新，将刷新缓存值的任务交给后台线程，所有的用户请求线程均返回旧的缓存值。

方法是覆盖CacheLoader的reload方法，使用线程池去异步加载数据

PS：只有重写了 reload 方法才有“异步加载”的效果。默认的 reload 方法就是同步去执行 load 方法。



#### 总结

大家都应该对各个失效/刷新机制有一定的理解，清楚在各个场景可以使用哪个配置，简单总结一下：

1. expireAfterWrite 是允许一个线程进去load方法，其他线程阻塞等待。

2. refreshAfterWrite 是允许一个线程进去load方法，其他线程返回旧的值。

3. 在上一点基础上做成异步，即回源线程不是请求线程。异步刷新是用线程异步加载数据，期间所有请求返回旧的缓存值。



## 实现原理

### 数据结构

Guava Cache的数据结构跟JDK1.7的ConcurrentHashMap类似，如下图所示：

![upload successful](/images/Guava Cache实现原理及最佳实践__0.png)

#### LoadingCache

LoadingCache即是我们API Builder返回的类型，类继承图如下：

![upload successful](/images/Guava Cache实现原理及最佳实践__1.png)



#### LocalCache

LoadingCache这些类表示获取Cache的方式，可以有多种方式，但是它们的方法最终调用到LocalCache的方法，LocalCache是Guava Cache的核心类。看看LocalCache的定义：

```java
class LocalCache<K, V> extends AbstractMap<K, V> implements ConcurrentMap<K, V>
```

说明Guava Cache本质就是一个Map。



LocalCache的重要属性：

```java
  //Map的数组
  final Segment<K, V>[] segments;
	//并发量，即segments数组的大小
  final int concurrencyLevel;
  //key的比较策略，跟key的引用类型有关
  final Equivalence<Object> keyEquivalence;
  //value的比较策略，跟value的引用类型有关
  final Equivalence<Object> valueEquivalence;
  //key的强度，即引用类型的强弱
  final Strength keyStrength;
  //value的强度，即引用类型的强弱
  final Strength valueStrength;
  //访问后的过期时间，设置了expireAfterAccess就有
  final long expireAfterAccessNanos;
  //写入后的过期时间，设置了expireAfterWrite就有
  final long expireAfterWriteNa就有nos;
  //刷新时间，设置了refreshAfterWrite就有
  final long refreshNanos;
  //removal的事件队列，缓存过期后先放到该队列
  final Queue<RemovalNotification<K, V>> removalNotificationQueue;
  //设置的removalListener
  final RemovalListener<K, V> removalListener;
  //时间器
  final Ticker ticker;
  //创建Entry的工厂，根据引用类型不同
  final EntryFactory entryFactory;
```



#### Segment

从上面可以看出LocalCache这个Map就是维护一个Segment数组。Segment是一个ReentrantLock

```java
static class Segment<K, V> extends ReentrantLock
```

看看Segment的重要属性：

```java
    //LocalCache
    final LocalCache<K, V> map;
    //segment存放元素的数量
    volatile int count;
    //修改、更新的数量，用来做弱一致性
    int modCount;
    //扩容用
    int threshold;
    //segment维护的数组，用来存放Entry。这里使用AtomicReferenceArray是因为要用CAS来保证原子性
    volatile @MonotonicNonNull AtomicReferenceArray<ReferenceEntry<K, V>> table;
    //如果key是弱引用的话，那么被GC回收后，就会放到ReferenceQueue，要根据这个queue做一些清理工作
    final @Nullable ReferenceQueue<K> keyReferenceQueue;
    //跟上同理
    final @Nullable ReferenceQueue<V> valueReferenceQueue;
    //如果一个元素新写入，则会记到这个队列的尾部，用来做expire
    @GuardedBy("this")
    final Queue<ReferenceEntry<K, V>> writeQueue;
    //读、写都会放到这个队列，用来进行LRU替换算法
    @GuardedBy("this")
    final Queue<ReferenceEntry<K, V>> accessQueue;
    //记录哪些entry被访问，用于accessQueue的更新。
    final Queue<ReferenceEntry<K, V>> recencyQueue;
```



#### ReferenceEntry

ReferenceEntry就是一个Entry的引用，有几种引用类型：

![upload successful](/images/Guava Cache实现原理及最佳实践__2.png)

我们拿StrongEntry为例，看看有哪些属性：

```java
    final K key;
    final int hash;
    //指向下一个Entry，说明这里用的链表（从上图可以看出）
    final @Nullable ReferenceEntry<K, V> next;
    //value
    volatile ValueReference<K, V> valueReference = unset();
```



### 源码分析

当我们了解了Guava Cache的结构后，那么进行源码分析就会简单很多。

本文只对put和get这两个重点操作来进行源码分析，其他源码如果读者感兴趣请自行阅读。

以下源码基于guava-26.0-jre版本。



#### get

##### get主流程

我们从LoadingCache的get(key)方法入手：

```java
//LocalLoadingCache的get方法，直接调用LocalCache
public V get(K key) throws ExecutionException {
      return localCache.getOrLoad(key);
    }

```

LocalCache：

```java
V getOrLoad(K key) throws ExecutionException {
    return get(key, defaultLoader);
  }

V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    //根据key获取hash值
    int hash = hash(checkNotNull(key));
    //通过hash定位到是哪个Segment，然后是Segment的get方法
    return segmentFor(hash).get(key, hash, loader);
  }


```

Segment：

```java
V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      checkNotNull(key);
      checkNotNull(loader);
      try {
        //这里是进行快速判断，如果count != 0则说明呀已经有数据
        if (count != 0) {
          //根据hash定位到table的第一个Entry
          ReferenceEntry<K, V> e = getEntry(key, hash);
          if (e != null) {
            //跟currentTimeMillis类似
            long now = map.ticker.read();
            //获取还没过期的value，如果过期了，则返回null。getLiveValue下面展开
            V value = getLiveValue(e, now);
            //Entry还没过期
            if (value != null) {
              //记录被访问过
              recordRead(e, now);
              //命中率统计
              statsCounter.recordHits(1);
              //判断是否需要刷新，如果需要刷新，那么会去异步刷新，且返回旧值。scheduleRefresh下面展开
              return scheduleRefresh(e, key, hash, value, now, loader);
            }
            
            ValueReference<K, V> valueReference = e.getValueReference();
            //如果entry过期了且数据还在加载中，则等待直到加载完成。这里的ValueReference是LoadingValueReference，其waitForValue方法是调用内部的Future的get方法，具体读者可以点进去看。
            if (valueReference.isLoading()) {
              return waitForLoadingValue(e, key, valueReference);
            }
          }
        }
        
        //重点方法。lockedGetOrLoad下面展开
        //走到这一步表示: 之前没有写入过数据 || 数据已经过期 || 数据不是在加载中。
        return lockedGetOrLoad(key, hash, loader);
      } catch (ExecutionException ee) {
        Throwable cause = ee.getCause();
        if (cause instanceof Error) {
          throw new ExecutionError((Error) cause);
        } else if (cause instanceof RuntimeException) {
          throw new UncheckedExecutionException(cause);
        }
        throw ee;
      } finally {
        postReadCleanup();
      }
    }

    //getLiveValue
    V getLiveValue(ReferenceEntry<K, V> entry, long now) {
      //被GC回收了
      if (entry.getKey() == null) {
        //
        tryDrainReferenceQueues();
        return null;
      }
      V value = entry.getValueReference().get();
      //被GC回收了
      if (value == null) {
        tryDrainReferenceQueues();
        return null;
      }
      //判断是否过期
      if (map.isExpired(entry, now)) {
        tryExpireEntries(now);
        return null;
      }
      return value;
    }

    //isExpired，判断Entry是否过期
    boolean isExpired(ReferenceEntry<K, V> entry, long now) {
      checkNotNull(entry);
      //如果配置了expireAfterAccess，用当前时间跟entry的accessTime比较
      if (expiresAfterAccess() && (now - entry.getAccessTime() >= expireAfterAccessNanos)) {
        return true;
      }
      //如果配置了expireAfterWrite，用当前时间跟entry的writeTime比较
      if (expiresAfterWrite() && (now - entry.getWriteTime() >= expireAfterWriteNanos)) {
        return true;
      }
      return false;
    }

    
```



##### scheduleRefresh

从get的流程得知，如果entry还没过期，则会进入此方法，尝试去刷新数据。

```java
   V scheduleRefresh(
        ReferenceEntry<K, V> entry,
        K key,
        int hash,
        V oldValue,
        long now,
        CacheLoader<? super K, V> loader) {
      //1、是否配置了refreshAfterWrite
      //2、用writeTime判断是否达到刷新的时间
      //3、是否在加载中，如果是则没必要再进行刷新
      if (map.refreshes()
          && (now - entry.getWriteTime() > map.refreshNanos)
          && !entry.getValueReference().isLoading()) {
        //异步刷新数据。refresh下面展开
        V newValue = refresh(key, hash, loader, true);
        //返回新值
        if (newValue != null) {
          return newValue;
        }
      }
      //否则返回旧值
      return oldValue;
    }

		//refresh
		V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
      //为key插入一个LoadingValueReference，实质是把对应Entry的ValueReference替换为新建的LoadingValueReference。insertLoadingValueReference下面展开
      final LoadingValueReference<K, V> loadingValueReference =
          insertLoadingValueReference(key, hash, checkTime);
      if (loadingValueReference == null) {
        return null;
      }
      //通过loader异步加载数据，这里返回的是Future。loadAsync下面展开
      ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
      //这里立即判断Future是否已经完成，如果是则返回结果。否则返回null。因为是可能返回immediateFuture或者ListenableFuture。
      //这里的官方注释是: Returns the newly refreshed value associated with key if it was refreshed inline, or null if another thread is performing the refresh or if an error occurs during
      if (result.isDone()) {
        try {
          return Uninterruptibles.getUninterruptibly(result);
        } catch (Throwable t) {
          // don't let refresh exceptions propagate; error was already logged
        }
      }
      return null;
    }

		//insertLoadingValueReference方法。
    //这个方法虽然看上去有点长，但其实挺简单的，如果你熟悉HashMap的话。
		LoadingValueReference<K, V> insertLoadingValueReference(
        final K key, final int hash, boolean checkTime) {
      ReferenceEntry<K, V> e = null;
      //把segment上锁
      lock();
      try {
        long now = map.ticker.read();
        //做一些清理工作
        preWriteCleanup(now);

        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);
        
        //如果key对应的entry存在
        for (e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          //通过key定位到entry
          if (e.getHash() == hash
              && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {
            
            ValueReference<K, V> valueReference = e.getValueReference();
            //如果是在加载中，或者还没达到刷新时间，则返回null
            //这里对这个判断再进行了一次，我认为是上锁lock了，再重新获取now，对时间的判断更加准确
            if (valueReference.isLoading()
                || (checkTime && (now - e.getWriteTime() < map.refreshNanos))) {
              
              return null;
            }
            
            //new一个LoadingValueReference，然后把entry的valueReference替换掉。
            ++modCount;
            LoadingValueReference<K, V> loadingValueReference =
                new LoadingValueReference<>(valueReference);
            e.setValueReference(loadingValueReference);
            return loadingValueReference;
          }
        }
        
        ////如果key对应的entry不存在，则新建一个Entry，操作跟上面一样。
        ++modCount;
        LoadingValueReference<K, V> loadingValueReference = new LoadingValueReference<>();
        e = newEntry(key, hash, first);
        e.setValueReference(loadingValueReference);
        table.set(index, e);
        return loadingValueReference;
      } finally {
        unlock();
        postWriteCleanup();
      }
    }

		//loadAsync
		ListenableFuture<V> loadAsync(
        final K key,
        final int hash,
        final LoadingValueReference<K, V> loadingValueReference,
        CacheLoader<? super K, V> loader) {
      //通过loadFuture返回ListenableFuture。loadFuture下面展开
      final ListenableFuture<V> loadingFuture = loadingValueReference.loadFuture(key, loader);
      //对ListenableFuture添加listener，当数据加载完后的后续处理。
      loadingFuture.addListener(
          new Runnable() {
            @Override
            public void run() {
              try {
                //这里主要是把newValue set到entry中。还涉及其他一系列操作，读者可自行阅读。
                getAndRecordStats(key, hash, loadingValueReference, loadingFuture);
              } catch (Throwable t) {
                logger.log(Level.WARNING, "Exception thrown during refresh", t);
                loadingValueReference.setException(t);
              }
            }
          },
          directExecutor());
      return loadingFuture;
    }

		//loadFuture
		public ListenableFuture<V> loadFuture(K key, CacheLoader<? super K, V> loader) {
      try {
        stopwatch.start();
        //这个oldValue指的是插入LoadingValueReference之前的ValueReference，如果entry是新的，那么oldValue就是unset，即get返回null。
        V previousValue = oldValue.get();
        //这里要注意***
        //如果上一个value为null，则调用loader的load方法，这个load方法是同步的。
        //这里需要使用同步加载的原因是，在上面的“缓存失效机制”也说了，即使用异步，但是还没有oldValue也是没用的。如果在系统启动时来高并发请求的话，那么所有的请求都会阻塞，所以给热点数据预加热是很有必要的。
        if (previousValue == null) {
          V newValue = loader.load(key);
          return set(newValue) ? futureValue : Futures.immediateFuture(newValue);
        }
        //否则，使用reload进行异步加载
        ListenableFuture<V> newValue = loader.reload(key, previousValue);
        if (newValue == null) {
          return Futures.immediateFuture(null);
        }
        
        return transform(
            newValue,
            new com.google.common.base.Function<V, V>() {
              @Override
              public V apply(V newValue) {
                LoadingValueReference.this.set(newValue);
                return newValue;
              }
            },
            directExecutor());
      } catch (Throwable t) {
        ListenableFuture<V> result = setException(t) ? futureValue : fullyFailedFuture(t);
        if (t instanceof InterruptedException) {
          Thread.currentThread().interrupt();
        }
        return result;
      }
    }
		

```



##### lockedGetOrLoad

如果之前没有写入过数据 || 数据已经过期 || 数据不是在加载中，则会调用lockedGetOrLoad

```java
		V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      ReferenceEntry<K, V> e;
      ValueReference<K, V> valueReference = null;
      LoadingValueReference<K, V> loadingValueReference = null;
      //用来判断是否需要创建一个新的Entry
      boolean createNewEntry = true;
      //segment上锁
      lock();
      try {
        // re-read ticker once inside the lock
        long now = map.ticker.read();
        //做一些清理工作
        preWriteCleanup(now);

        int newCount = this.count - 1;
        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        //通过key定位entry
        for (e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          if (e.getHash() == hash
              && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {
            //找到entry
            valueReference = e.getValueReference();
            //如果value在加载中则不需要重复创建entry
            if (valueReference.isLoading()) {
              createNewEntry = false;
            } else {
              V value = valueReference.get();
              //value为null说明已经过期且被清理掉了
              if (value == null) {
                //写通知queue
                enqueueNotification(
                    entryKey, hash, value, valueReference.getWeight(), RemovalCause.COLLECTED);
              //过期但还没被清理
              } else if (map.isExpired(e, now)) {
                //写通知queue
                // This is a duplicate check, as preWriteCleanup already purged expired
                // entries, but let's accomodate an incorrect expiration queue.
                enqueueNotification(
                    entryKey, hash, value, valueReference.getWeight(), RemovalCause.EXPIRED);
              } else {
                recordLockedRead(e, now);
                statsCounter.recordHits(1);
                //其他情况则直接返回value
                //来到这步，是不是觉得有点奇怪，我们分析一下: 
                //进入lockedGetOrLoad方法的条件是数据已经过期 || 数据不是在加载中，但是在lock之前都有可能发生并发，进而改变entry的状态，所以在上面中再次判断了isLoading和isExpired。所以来到这步说明，原来数据是过期的且在加载中，lock的前一刻加载完成了，到了这步就有值了。
                return value;
              }
              
              writeQueue.remove(e);
              accessQueue.remove(e);
              this.count = newCount; // write-volatile
            }
            break;
          }
        }
        //创建一个Entry，且set一个新的LoadingValueReference。
        if (createNewEntry) {
          loadingValueReference = new LoadingValueReference<>();

          if (e == null) {
            e = newEntry(key, hash, first);
            e.setValueReference(loadingValueReference);
            table.set(index, e);
          } else {
            e.setValueReference(loadingValueReference);
          }
        }
      } finally {
        unlock();
        postWriteCleanup();
      }
			//同步加载数据。里面的方法都是在上面有提及过的，读者可自行阅读。
      if (createNewEntry) {
        try {
          synchronized (e) {
            return loadSync(key, hash, loadingValueReference, loader);
          }
        } finally {
          statsCounter.recordMisses(1);
        }
      } else {
        // The entry already exists. Wait for loading.
        return waitForLoadingValue(e, key, valueReference);
      }
    }
```



##### 流程图

通过分析get的主流程代码，我们来画一下流程图：

![upload successful](/images/Guava Cache实现原理及最佳实践__3.png)



#### put

看懂了get的代码后，put的代码就显得很简单了。

Segment的put方法:

```java
		V put(K key, int hash, V value, boolean onlyIfAbsent) {
      //Segment上锁
      lock();
      try {
        long now = map.ticker.read();
        preWriteCleanup(now);

        int newCount = this.count + 1;
        if (newCount > this.threshold) { // ensure capacity
          expand();
          newCount = this.count + 1;
        }

        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        //根据key找entry
        for (ReferenceEntry<K, V> e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          if (e.getHash() == hash
              && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {

            //定位到entry
            ValueReference<K, V> valueReference = e.getValueReference();
            V entryValue = valueReference.get();
            //value为null说明entry已经过期且被回收或清理掉
            if (entryValue == null) {
              ++modCount;
              if (valueReference.isActive()) {
                enqueueNotification(
                    key, hash, entryValue, valueReference.getWeight(), RemovalCause.COLLECTED);
                //设值
                setValue(e, key, value, now);
                newCount = this.count; // count remains unchanged
              } else {
                setValue(e, key, value, now);
                newCount = this.count + 1;
              }
              this.count = newCount; // write-volatile
              evictEntries(e);
              return null;
            } else if (onlyIfAbsent) {
              //如果是onlyIfAbsent选项则返回旧值
              recordLockedRead(e, now);
              return entryValue;
            } else {
              //不是onlyIfAbsent，设值
              ++modCount;
              enqueueNotification(
                  key, hash, entryValue, valueReference.getWeight(), RemovalCause.REPLACED);
              setValue(e, key, value, now);
              evictEntries(e);
              return entryValue;
            }
          }
        }
        
        //没有找到entry，则新建一个Entry并设值
        ++modCount;
        ReferenceEntry<K, V> newEntry = newEntry(key, hash, first);
        setValue(newEntry, key, value, now);
        table.set(index, newEntry);
        newCount = this.count + 1;
        this.count = newCount; // write-volatile
        evictEntries(newEntry);
        return null;
      } finally {
        unlock();
        postWriteCleanup();
      }
    }
```

put的流程相对get来说没有那么复杂。



## 最佳实践

关于最佳实践，在上面的“缓存失效机制”中得知，看来使用refreshAfterWrite是一个不错的选择，但是从上面get的源码分析和流程图看出，或者了解Guava Cache都知道，Guava Cache是没有定时器或额外的线程去做清理或加载操作的，都是通过get来触发的，目的是降低复杂性和减少对系统的资源消耗。

那么只使用refreshAfterWrite或配置不当的话，会带来一个问题：如果一个key很长时间没有访问，这时来一个请求的话会返回旧值，这个好像不是很符合我们的预想，在并发下返回旧值是为了不阻塞，但是在这个场景下，感觉有足够的时间和资源让我们去刷新数据。

结合get的流程图，在get的时候，是先判断过期，再判断refresh，即如果过期了会优先调用 load 方法（阻塞其他线程），在不过期情况下且过了refresh时间才去做 reload （异步加载，同时返回旧值），所以推荐的设置是 refresh < expire，这个设置还可以解决一个场景就是，如果长时间没有访问缓存，可以保证 expire 后可以取到最新的值，而不是因为 refresh 取到旧值。

用一张时间轴图简单表示：


![upload successful](/images/Guava Cache实现原理及最佳实践__4.png)



## 总结

Guava Cache是一个很优秀的本地缓存工具，缓存的作用不多说，一个简单易用，功能强大的工具会使你在开发中事倍功半。但是跟所有的工具一样，你要在了解其内部原理、机制的情况下，才能发挥其最大的功效，才能适用到你的业务场景中。

本文通过对Guava Cache的使用、核心机制的讲解、核心源代码的分析以及最佳实践的说明，相信你会对Guava Cache有更进一步的了解。

