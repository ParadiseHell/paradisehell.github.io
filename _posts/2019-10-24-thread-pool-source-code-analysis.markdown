---
layout:     post
title:      "线程池源码分析"
subtitle:   "彻底搞清楚线程池的原理"
date:       2019-10-24
author:     "ChengTao"
header-img: "img/linux.jpg"
tags:
    - Java
    - 源码
---

> 如果你使用 Google 搜索 “Java 线程池源码解析” 你将会搜索到无数相关的博客，这里我推荐这篇博客 <a href="http://gityuan.com/2016/01/16/thread-pool/" target="_blank">Java线程池分析(Gityuan)</a>。不过大多数博客的通病就是一上来就是描述为什么要使用线程池，接下来就是各种源码分析；这种显然不够优雅，真正优雅的源码分析应该是带着问题一步一步的揭开源码的神秘面纱，最后分析完后再回头看问题，便有一种豁然开朗的体验。

### 问题（思考）列表
以下是我认为大家需要思考的一些问题：
- 为什么线程不能执行两次 start ?
- 既然线程不能执行多次 start, 哪些线程池是如何实现线程复用的？
- 线程池提供了 corePoolSize 这个参数，那么线程池是如何保证 corePoolSize 个线程的存在的？
- 线程池提供了 maximumPoolSize 这个参数，那么线程池是否真的会创建 maximumPoolSize 个线程吗？

### 环境说明
- 基于 Java JDK 1.8

### 为什么线程不能执行两次 start ?

直接看源码:
```java
public synchronized void start() {
	if (threadStatus != 0)
		throw new IllegalThreadStateException();
	group.add(this);
	boolean started = false;
	try {
		start0();
		started = true;
	} finally {
		try {
			if (!started) {
				group.threadStartFailed(this);
			}
		} catch (Throwable ignore) {
		}
	}
}
```
从源码中可以看出，当执行 start 的时候, 会判断 **threadStatus** 是否为 0, 如果不为 0, 则会抛出异常；而第一次 start 后，**threadStatus** 会在 native 层被设置为 RUNNABLE(1) (具体是如何设置的可以查看 OpenJDK 的相关 native 源码), 所以第二次之后再调用 start 的时候则会抛出异常。

所以线程不可以多次执行 start 方法，那么线程池是如何进行线程复用呢？接下来进行最重要的线程池源码分析。

### ThreadPoolExecutor 源码分析

##### 阅读源码准备工作

1. **ThreadPoolExecutor** 的构造函数:
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
- corePoolSize : 线程池核心线程数量，必须**大于等于** 0.
- maximumPoolSize : 线程池最大线程数量，必须**大于等于** 1 并且**大于等于** corePoolSize.
- keepAliveTime : 线程保活时间，必须**大于等于** 0.
- unit : 线程保活时间的单位.
- workQueue : 工作队列，必须是阻塞队列。
- threadFactory : 线程工厂，用于创建线程，默认值为 **Executors.defaultThreadFactory()**;
- handler : 拒绝策略，用于处理提交任务数量大于 maximumPoolSize 时，对任务的处理，默认值为 **new AbortPolicy()**.
2. 几个关键的属性:
- ctl(AtomicInteger) : 线程池控制状态，一个被两个字段 **workerCount** (工作线程数量)和 **runState** (线程池运行状态)打包成的原子 Integer.
		- workerCount : 存储在 Integer 的后 29 位，所以 workerCount 的最大值是 **2^29 - 1**(大概 5 亿)，而不是 **2^32 - 1**(大概 20 亿).
		- runState 存储在 Integer 的前 3 位，有 **RUNNING**, **SHUTDOWN**, **STOP**, **TIDYING** 和 **TERMINATED** 共 5 个状态。
3. 线程池状态说明
- RUNNING : 接收新的任务、处理任务队列里的任务。
- SHUTDOWM : 不接收新的任务、处理任务队列里的任务。
- STOP : 不接收新的任务、不处理任务队列里的任务、打断正在运行的任务。
- TIDYING : 所有任务已经完成，workerCount 为 0, 线程池状态转为 TIDYING, 并执行 terminated 方法。
- TERMINATED : terminated 方法执行完毕。

##### execute 方法源码分析

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // 处理 command 一共 3 个步骤:
        // 1. 如果当前线程池中正在运行的线程数小于 corePoolSize, 尝试创建一个新的
        // 线程来处理 command, 并调用 addWorkder 方法来处理 command, addWorker 方法
        // 会自动判断 runState 和 workerCount, 通过返回 false 提示不能创建新的线程
        // 来处理此 command.
        int c = ctl.get();
        // 判断 workerCount 和 corePoolSize 大小
        if (workerCountOf(c) < corePoolSize) {
        // 添加新的 worker 处理 command
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2. 如果线程池中正在运行的线程大于 corePoolSize, 先判断线程池是否处于 
        // RUNNING 状态，之后将 command 插入队列，如果插入成功，会进行二次检测线程池。
        //
        //----1). 是否需要创建新的线程处理此 command (调用 addWorker 方法):
        //--------I. 存在线程池中只存在一个线程，当将 command 插入队列后就处于 
        //--------ThreadState.TERMINATED 状态，不能再处理 command.
        //
        //--------II. 存在线程池中就没有任何线程，当前 corePoolSize == 0 并且第一次
        //--------执行 execute 方法处理此 command.
        //
        //----2). 是否需要回滚 command :
        //--------I. 存在插入 command 到队列后，线程池状态不为 RUNNING, 这时候就需要
        //--------将 command 移除队列，并执行 reject 方法。
		
        // 判断线程池状态并将 command 插入队列
       	if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 检测线程池状态并回滚 command
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 创建新的线程处理 command
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }

        // 3. 如果 command 无法插入队列，尝试创建新的线程处理 command, 如果失败，表示
        // 线程池已经不处于 RUNNING 状态或者线程池处于饱和状态，并执行 reject 方法拒
        // 绝 command.

        // 尝试创建新的线程处理 command
        else if (!addWorker(command, false))
        // 拒绝 command
            reject(command);
    }
```

##### reject 方法源码分析

```java
    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```

因为默认的 handler 为 AbortPolicy, 所以这里查看 AbortPolicy 的源码 :

```java
    public static class AbortPolicy implements RejectedExecutionHandler {

        public AbortPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```
可以看出默认情况，如果线程拒绝 command 则会抛出一个 **RejectedExecutionException** 异常, 所以如果我们不想出现这个异常则可以继承 **RejectedExecutionHandler**, 然后进行相关处理，例如重试等，不过不要忘记判断线程池的状态，不然可能出现死亡调用，并出现 **StackOverflowError**.

##### addWorker 方法源码分析

```java
    // firstTask : 新线程要处理的第一的任务
    // 
    // core : 如果为 true, 使用 coolPoolSize 作为边界处理，否则使用 maximumPoolSize
    // 处理
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        // LOOP1
        for (;;) {
            int c = ctl.get();
            // 获取线程池状态
            int rs = runStateOf(c);

            // 判断线程池状态、工作队列是否是空的
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            // LOOP2
            for (;;) {
                // 获取工作线程数(workerCount)
                int wc = workerCountOf(c);
                // 判断 workerCount 是否大于最大容量（2^29 - 1）或者通过 core 判
                // 断是否超过边界
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 累加 workCount, 如果累加成功，跳出两个循环 LOOP1 和 LOOP2
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 重新检测状态，如果当前获取的状态和最新状态不相同跳到 LOOP1
                c = ctl.get(); 
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

        // 标记 worker 是否启动
        boolean workerStarted = false;
        // 标记 worker 是否被添加
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建 Worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            // 判断线程是否为 null, 因为线程是由 ThreadFactory 创建的，存在自定
            // 义 ThreadFactory 导致线程为 null.
            if (t != null) {
                // 上锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 重新检测线程池状态
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 如果线程处于 alive 状态，即线程已经执行了 start 方法
                        // 抛出 IllegalThreadStateException 异常，由于线程是由
                        // ThreadFactory 创建的，可能由于自定义导致使用了已经
                        // start 过的线程
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();

                        // 添加 worker 
                        workers.add(w);
                        // 记录最大线程池大小
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 标记 worker 已经被添加
                        workerAdded = true;
                    }
                } finally {
                    // 释放锁
                    mainLock.unlock();
                }

                // 判断 worker 是否被添加
                if (workerAdded) {
                    // 启动线程
                    t.start();
                    // 标记 worker 被启动
                    workerStarted = true;
                }
            }
        } finally {
            // 如果 worker 没有启动过，处理 worker 添加失败
            if (! workerStarted)
                addWorkerFailed(w);
        }
        // 返回 worker 是否被启动
        // true : 添加 worker 成功；false : 添加 worker 失败
        return workerStarted;
    }
```

##### Worker 源码分析

```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        // 序列化 UID
        private static final long serialVersionUID = 6138294804551838833L;

        // 当前 Thead 需要首先执行的 task
        Runnable firstTask;

        // 上一个线程的完成的任务数
        volatile long completedTasks;

        // firstTask : 第一个执行的任务, 可能为 null
        Worker(Runnable firstTask) {
            // 防止再执行 runWorker 前被打断
            setState(-1); 
            this.firstTask = firstTask;
            // 创建线程，并传入自己( Runnable )作为参数
            this.thread = getThreadFactory().newThread(this);
        }

        // 代理 run 方法
        public void run() {
            runWorker(this);
        }

        // 锁相关方法
        //
        // 0 表示未上锁状态
        // 1 表示上锁状态

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
**Worker** 继承 **AbstractQueuedSynchronizer**, 所以 **Worker** 可以实现同步操作；又由于实现了 **Runnable** 所以**Worker** 在创建线程的时候把自己作为参数传进去了，在上一步 **Thead.start()** 的时候，会调用 **Worker** 的 **run** 方法，进入调用 runWorker 方法。

##### runWorker 方法源码分析

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        // task 记录当前任务
        Runnable task = w.firstTask;
        w.firstTask = null;
        // 释放锁，允许 Worker 被打断
        w.unlock();
        boolean completedAbruptly = true;
        try {
            // 一直获取 task, 直到 task 为 null, 跳出循环
            while (task != null || (task = getTask()) != null) {
                // 上锁
                w.lock();
                // 如果线程池的状态为 STOP, 保证当前线程被打断过
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 执行 task 前, 可以在此处做 hook 相关操作 
                    beforeExecute(wt, task);
                    // 记录异常
                    Throwable thrown = null;
                    try {
                        // 具体执行任务，也就是 execute 方法中的 command
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 执行 task 后, 可以在此处做 hook 相关操作
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 置空 task
                    task = null;
                    // 累加 worker 完成的任务数
                    w.completedTasks++;
                    // 释放锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 处理 worker 退出
            processWorkerExit(w, completedAbruptly);
        }
    }
```

其实从 runWorker 函数我们已经揭开了线程池复用的面纱，其实就是创建 Thread 的时候使用 Runnable 作为参数, 所以 Thread 的 run 方法会执行 Runnable 的 run 方法，在这里就是 Worker 的 run 方法，而这个 run 方法中是一个 while 死循环不停的获取 task(Runnable), 然后执行 runable 的 run 方法，所以复用线程没有黑科技，就是相当于在线程的 run 方法中执行死循环，保证线程一直处于 RUNNABLE 状态。

##### getTask 方法源码分析

```java
    private Runnable getTask() {
        // 标记是否超时
        boolean timedOut = false; // Did the last poll() time out?
        // 死循环
        for (;;) {
            // 获取线程池状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 判读线程池状态是否大于等于 SHUTDOWN
            // 再线程池状态是否大于等于 STOP 或 判断任务队列是否为空
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                // 递减 workerCount
                decrementWorkerCount();
                // 返回 null, 表示没有更多的任务
                return null;
            }

            // 获取 workerCount
            int wc = workerCountOf(c);

            // 判断是否使用超时获取任务
            // 1. allowCoreThreadTimeOut : 可以通过 Thread#allowCoreThreadTimeOut 设置
            // 2. 如果 workerCount 大于 coolPoolSize 也进行超时获取任务
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 如果 workerCount 大于 maximumPoolSize 或者获取任务超时
            // 并且 workerCount  > 1 或者 任务队列为空
            //
            // workerCount 大于 maximumPoolSize 基本不可能，所以主要判断是否超时
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                // 判断递减 workerCount 是否成功
                // 成功返回 null 表示没有更多任务, 失败继续死循环
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 判断是否使用超时从任务队列获取任务
                // poll 方法在规定时间没有获取任务返回 null
                // take 方法则是一直阻塞直到获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                // 如果获取的任务不为 null, 返回任务
                if (r != null)
                    return r;
                // 标记超时
                timedOut = true;
            } catch (InterruptedException retry) {
                // 如果出现异常，重置超时标志
                timedOut = false;
            }
        }
    }
```

从 getTask 方法的作用是从从任务队列中获取任务，而这里面有一个小技巧，保证了线程池始终存在 corePoolSize 数量的线程(allowCoreThreadTimeOut 为 true 除外)，就是当 workerCount > corePoolSize 的时候，默认变成超时从任务队列获取任务，这样如果没有更多的任务，getTask 方法就会因为获取任务超时返回 null, 进而导致 runWorker 方法结束死循环，则当前线程将结束 RUNNABLE 状态，最终到达 TERMINATED 状态。

##### processWorkerExit 方法源码分析

```java
    // w : worker 实例
    // completedAbruptly : 是否 worker 是因为用户异常导致死亡的
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果 worker 是因为用户异常导致死亡的，递减 workerCount
        if (completedAbruptly)
            decrementWorkerCount();
        
        // 上锁移除 worker
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        // 尝试结束线程池
        tryTerminate();

        // 判断当前线程池的状态，如果线程池状态小于 STOP, 即线程池状态为 RUNNABLE
        // 或者 SHUTDOWN, 这两个状态仍然要处理任务队列里的任务。
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            // 如果正常退出
            if (!completedAbruptly) {
                // 判断当前线程池是否有足够的线程存在
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果有足够线程存在, 直接返回
                if (workerCountOf(c) >= min)
                    return; 
            }
            // 添加 worker 保证有足够线程处理任务队列里的任务
            addWorker(null, false);
        }
    }
```

##### tryTerminate 方法源码分析

```java
    final void tryTerminate() {
        // 死循环
        for (;;) {
            // 判断线程池状态
            // 如果状态为 STOP 或者 SHUTDOWM 并且任务队列为空继续处理
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;

            // 判断 workerCount 是否为 0
            if (workerCountOf(c) != 0) { 
                // 打断没有任务的 Worker
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            
            // 上锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 设置线程池状态为 TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 执行 terminated 方法，默认为空，可以自定义
                        terminated();
                    } finally {
                        // 设置线程池状态为 TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
        }
    }
```

### 线程池执行任务策略

![](/img/2019/thread_pool_execute_task_policy.jpg)

### 问题回顾

截止目前我们还差最后一个问题没有回答，线程池提供了 maximumPoolSize 这个参数，那么线程池是否真的会创建 maximumPoolSize 个线程吗？ 

从上面的策略图可以看出线程池中的线程数要达到 maximumPoolSize 的条件是任务队列满了，如果任务队列不满，则线程池中永远只可能有 corePoolSize 个线程, 比如我们使用了 LinkedBlockingQueue, 则会有这样的效果。

### 线程池创建

可以使用 Executors 提供的方法进行创建可以非常轻松的创建符合我们需求的线程池:

1. 创建固定核心线程数的线程池:
- newFixedThreadPool(int nThreads) 	
		- nThreads : 核心线程数量
- newFixedThreadPool(int nThreads, ThreadFactory threadFactory)
		- nThreads : 核心线程数量
		- threadFactory : 线程工厂
2. 创建只有一个核心线程数的线程池:
- newSingleThreadExecutor()
- newSingleThreadExecutor(ThreadFactory threadFactory)
		- threadFactory : 线程工厂
3. 创建具有缓存的线程池:
- newCachedThreadPool()
- newCachedThreadPool(ThreadFactory threadFactory)
		- threadFactory : 线程工厂

### 线程池优点

- 降低系统资源消耗，通过重用已存在的线程，降低线程创建和销毁造成的消耗；
- 提高系统响应速度，当有任务到达时，无需等待新线程的创建便能立即执行；
- 方便线程并发数的管控，线程若是无限制的创建，不仅会额外消耗大量系统资源，更是占用过多资源而阻塞系统或 oom 等状况，从而降低系统的稳定性。线程池能有效管控线程，统一分配、调优，提供资源使用率；
- 更强大的功能，线程池提供了定时、定期以及可控线程数等功能的线程池，使用方便简单。

### 总结

希望这篇源码可以帮助你更好的理解线程池的原理。
