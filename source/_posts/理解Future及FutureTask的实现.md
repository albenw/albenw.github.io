title: 理解Future及FutureTask的实现
author: alben.wong
abbrlink: 5dd69f23
tags:
  - future
  - futuretask
  - 异步
categories:
  - java
keywords: future futuretask
description: Future是一种异步计算的模式，本文带你理解一下什么是Future，以及基本的FutureTask的实现原理。
date: 2019-02-15 17:09:00
---
## 概要
Future是一种异步计算的模式，本文带你理解一下什么是Future，以及基本的FutureTask的实现原理。


## 作用
如果在一个方法中要执行另一个操作（任务），但是这个操作会耗时很久，而且你后面还需要用到这个操作的返回结果或者必须等到这个操作结束你才能走下去，你会怎样做？可能大家都会想到异步去执行，即新建一个线程去做这个事情，但是这样的话，你后面的操作就要放到这个异步线程那里，你的方法就变成异步的了，对你原来的返回造成了影响。  
这时候，Future就发挥作用了，有些地方说它是一种模式，其实，它就是对一个异步操作的封装，它会返回一个“凭证”给你，你可以用这个“凭证”在需要的时候获取到这个异步操作的结果，一般来说这个“凭证”就是future。


## 原理
FutureTask就是Future的基本实现，下面我们就从代码分析一下实现的原理。   
源码基本JDK1.8。


### Future接口
我们先看看Future的定义，即你拿到这个“凭证”之后你能干点什么。
```java
public interface Future<V> {

	//取消这次任务
    boolean cancel(boolean mayInterruptIfRunning);
	//看看是否取消了
    boolean isCancelled();
	//看看是否完成了
    boolean isDone();
	//获取结果
    V get() throws InterruptedException, ExecutionException;
	//时间内获取结果，超时则抛异常
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```


### 使用
如果没用过的，这里简单演示一下Future怎么使用，大家有个感性认识。
```java
	private static ExecutorService threadPool = Executors.newCachedThreadPool();
    
    public static void main(String[] args) throws Exception {
    	//通过线程池提交任务，并返回一个future
        Future<String> future =  threadPool.submit(new AsyncTask());
        //通过future获取结果。get之前一直阻塞直到有结果返回。
        String result = future.get();
        System.out.println(result);
    }
    
    public class AsyncTask implements Callable<String> {
    
        @Override
        public String call() throws Exception {
            TimeUnit.SECONDS.sleep(5);
            return "ok";
        }
    }
```


### 继承关系

![upload successful](/images/理解Future及FutureTask的实现_0.png)

FutureTask是一个RunnableFuture，这个很好理解，就是Runnable\+Future了。


### 提交任务
从任务的提交入手分析源码。  
AbstractExecutorService的submit方法。
```java
public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        //这里封装了一下callable变成一个RunnableFuture
        RunnableFuture<T> ftask = newTaskFor(task);
        //注意线程池执行的是RunnableFuture，因为这个Future继承Runnable，所以它是可执行的。
        execute(ftask);
        //返回future
        return ftask;
    }
```

submit方法也是支持Runnable的。ExecutorService内部会把Runnable转成Callable，只不过Runnable的返回值为null。



### FutureTask的属性/状态
先看下FutureTask的一些内部属性，才好了解它是怎么运行的。
```java
	//重要属性“状态”state定义为volatile，为了在高并发下能够获取到最新的值
	private volatile int state;
    //为state定义了7个状态，看名字都挺好了解的。
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    //正常结束
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
    
    //状态之间的转换
    * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
     
    //用户提交的真正任务
    private Callable<V> callable;
   	//返回结果
    private Object outcome; // non-volatile, protected by state reads/writes
    //记录跑任务的那个线程，只有一个在运行。取消时可以中断这个线程的行为。
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    //这个属性比较重要，后面讲的比较多。记录WaitNode链表的头部，volatile。WaitNode链表是等待结果的线程集合，即这个任务还没跑完时，但同时有很多持有这个future的线程调用了get方法获取结果。
    private volatile WaitNode waiters;
    
    
```


### run方法
当这个RunnableFuture提交到线程池后，它做了什么。

```java
public void run() {
	    //为了防止future重复运行，需要判断是NEW状态。
        //同时记录runner属性
        //这里UNSAFE的CAS操作在JUC里用的比较多，不展开了（我的原则是，不是这个话题的内容不过多展开，保持专注和精简。）
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                	//执行用户的callable
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //如果callable运行异常了，这里会把这个异常吃掉，然后调用setException方法
                    setException(ex);
                }
                if (ran)
                	//正常结束就是设置结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

值得注意的是，如果原来的callable任务运行异常了，那么在run方法中会直接catch掉，然后在get的时候才抛出来。这么也是为了做错误隔离，为了callable的异常不会影响到future的运行。


#### setException方法
```java
protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        	//把结果设为异常
            outcome = t;
            //更新状态为EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            //finishCompletion作为一个结束动作
            finishCompletion();
        }
    }
```

#### set方法
```java
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        	//设置结果
            outcome = v;
            //更新状态为NORMAL
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
            //同样调用了finishCompletion方法
            finishCompletion();
        }
    }
```

setException方法和set方法都是protected，不能随意调用，不过子类可以改变它的行为。  

#### COMPLETING状态？  
在上面说到的7个状态中有一个COMPLETING的状态，它表示新建NEW和正常结束NORMAL或异常结束EXCEPTIONAL中间的这么一个状态，在set和setException用到了，会先把NEW状态更新为COMPLETING，再把COMPLETING更新为对应的结果状态。  
刚开始我认为这个状态是没必要的。因为这个FutureTask只会有一个线程在运行它，不存在竞争，而且看代码也知道，作者没对竞争失败做处理，那么set和setException的CAS操作是肯定会成功的，所以我觉得把COMPLETING变成NEW也是可以的。但是细想如果直接把
```java
UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)
```
变成
```java
UNSAFE.compareAndSwapInt(this, stateOffset, NEW, NORMAL)
```

但是后面还有一步赋值操作。
```java
outcome = v
```
并发下，这时如果有其他线程在CAS后想获取结果，就返回null了。


finishCompletion方法
```java
//如果这个future正常结束，异常结束，被取消了，都会调用这个方法。
private void finishCompletion() {
        // assert state > COMPLETING;
        //这里会不停的拿头部节点做遍历，直到头部节点为null。这是为了防止在并发下有新的节点新插入进来。
        for (WaitNode q; (q = waiters) != null;) {
        	//CAS把WaitNode链表的头部设为null。
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        //唤醒WaitNode的线程，唤醒的线程继续在awaitDone方法里做循环。
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    //直到next为null，完成链表的遍历
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        //这个done方法是预留方法，子类可以继承它来做点别的。
        done();

        callable = null;        // to reduce footprint
    }
```


### get方法
get方法是重点。  
因为作者设计FutureTask是支持高并发的，而且用了Lock-Free无锁算法，所以阅读起来会比较费劲。
```
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```

#### WaitNode链表
继续看下去之前，先看看WaitNode的定义。  
很简单，只有一个Thread和next指针。Thread就是指当前需要获取future结果的那个线程。WaitNode通过next指针形成一条链表。

```java
static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

这是在Lock-Free中常见的数据结构，看上去是不是有点像AQS呢？
```
/*
     * Revision notes: This differs from previous versions of this
     * class that relied on AbstractQueuedSynchronizer, mainly to
     * avoid surprising users about retaining interrupt status during
     * cancellation races. Sync control in the current design relies
     * on a "state" field updated via CAS to track completion, along
     * with a simple Treiber stack to hold waiting threads.
     *
     * Style note: As usual, we bypass overhead of using
     * AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
     */
```

实际上，官方也说了，之前版本的实现是用了AQS的，原因是...（这个原因我不是很懂是啥意思），现在改为Treiber stack算法了。


#### awaitDone方法
```java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        //死循环直到s > COMPLETING或者超时，当然这个不是真的死循环，大部分情况下线程是会挂起的。
        for (;;) {
        	//如果线程是被中断了，则从链表移除当前节点，然后抛异常
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            //从上面7个状态看出，当s > COMPLETING都是结束的状态，要不正常结束，异常，取消等。可见合理的状态值设计带来的方便。
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //这里就是为了防止上面我说的，结果赋值时并发下其他线程获取不到值的情况，所以让这个线程yield一下，再做一次循环，说不定下次就是s > COMPLETING呢。
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
            	//新建一个WaitNode，准备进链表
                q = new WaitNode();
            else if (!queued)
            	//CAS把WaitNode节点插入到链表的头部，如果失败则下次继续插入
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
             //如果get方法设置了超时时间，则会进入这个分支，如果超时了，也会返回state。还没超时则挂起，挂起的时间为时间差。
             else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                	//超时需要从链表移除当前节点
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
             }
             //注意这里，是最后的步骤，即当s < COMPLETING，已经插入链表了，不是超时的情况
             else
                LockSupport.park(this);
        }
    }
```

awaitDone方法返回的是state状态的值。  
要注意if else执行的顺序，先是判断中断状态，其次判断state的完成状态，再是新建节点，然后插入，最后才是挂起。用for循环去尝试当CAS失败的情况。

插入节点的只有这个方法，所以我们可以知道，链表的结构如下：

![upload successful](/images/理解Future及FutureTask的实现_1.png)


removeWaiter方法的作用是当中断或超时时移除当前的WaitNode。这个方法有点不好理解。
```java
private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```

我们画个图，分情况来理解一下

![upload successful](/images/理解Future及FutureTask的实现_2.png)

1. 如果q.thread != null  
因为进来时已经直接把node.thread = null，说明q已经不是当前的node，q是其他线程插入进来的node，这时需要把s，q，pred继续往左移动。
2. 如果q.thread == null && pred != null  
这时可以把pred的next指向s了，即删除了q。但是如果pred.thread == null，说明pred的线程也把它自己的节点删除了（删除节点的情况除了removeWaiter，还有正常获取结果后也会），所以pred已经没用了，需要重新来找到新的pred。
3. 如果q.thread == null && pred == null  
说明前面的节点都被删除了，已经没用了，把s直接置为头部。


#### report方法
比较简单了
```java
private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
        	//task正常执行就返回结果
          	return (V)x;
        if (s >= CANCELLED)
            //取消则抛异常
            throw new CancellationException();
        //否则抛出task运行的异常
        throw new ExecutionException((Throwable)x);
    }
```


### cancel方法

```java
//mayInterruptIfRunning参数，取消的同时可以中断runner线程的运行。
public boolean cancel(boolean mayInterruptIfRunning) {
		//状态更新为INTERRUPTING或CANCELLED
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
        	//中断
          	if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                    	//调用线程的interrupt方法来中断线程
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
        	//取消也要把watiers清空掉
            finishCompletion();
        }
        return true;
    }
```

## 缺点
FutureTask有明显的下面两个缺点：

1. 重复提交  
并发会有重复提交的可能，虽然在内部有对状态NEW的判断，但那只是针对那个FutureTask实例的，我们看到，在submit方法中每次提交任务都会new 一个
FutureTask出来的。  
不过现在已经有一个解决方案[Memoizer](http://jcip.net/listings/Memoizer.java)  
其实很简单，就是用一个key来记录这次的Future，然后放在一个Map里，下次用到时再从Map里取出来。


2. 批量任务  
Future每次只能提交一个任务，而且获取结果之前会一直阻塞，这点也是很不友好的。


综上，FutureTask只是提供了一个基本的功能实现，远远不能满足要求高的我们，guava的ListenableFuture和JDK1.8的CompletableFuture都是对Future的增强，前者提供监听器处理结果，后者更加强大，提供链式调用，同步、异步结果返回不同的组合方式来帮助你处理复杂的业务场景。



## 总结
源码部分已经介绍的7788了。因为采用了无锁算法，所以实现起来看上去代码比较复杂，看代码时要意识到这个，多想想在高并发下链表会出现怎样的情况，我没有把所有可能出现的情况都罗列出来，所以要靠读者自己多思考。  

总的来说，Future通过循环判断state状态，挂起、唤醒线程的操作，来实现异步阻塞，通过一个WaitNode链表来处理并发的情况。