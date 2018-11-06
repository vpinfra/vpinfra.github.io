---
id: 201811010908
title: ThreadPoolExcutor 源码解析
date: 2018-11-01 09:08:45
categories: 源码系列
tags: [源码系列]
author: harvey
---
### 常用的构造方法
- 构造函数一
```
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
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}

#### 构造方法参数说明：

corePoolSize：核心线程数，默认情况下核心线程会一直存活，即使处于闲置状态也不会受存keepAliveTime限制。除非将allowCoreThreadTimeOut设置为true。

maximumPoolSize：线程池所能容纳的最大线程数。超过这个数的线程将被阻塞。当任务队列为没有设置大小的LinkedBlockingDeque时，这个值无效。

keepAliveTime：非核心线程的闲置超时时间，超过这个时间就会被回收。

unit：指定keepAliveTime的单位，如TimeUnit.SECONDS。当将allowCoreThreadTimeOut设置为true时对corePoolSize生效。

workQueue：线程池中的任务队列.常用的有三种队列。
- SynchronousQueue：是一种无缓冲的等待队列，在某次添加元素后必须等待其他线程取走后才能继续添加；
- LinkedBlockingDeque：是一个无界缓存的等待队列，不指定容量则为Integer最大值，锁是分离的；
- ArrayBlockingQueue：是一个有界缓存的等待队列，必须指定大小，锁是没有分离的；

threadFactory：线程工厂，提供创建新线程的功能，通过线程工厂可以对线程的一些属性进行定制。

RejectedExecutionHandler：当线程池中的资源已经全部使用，添加新线程被拒绝时，会调用RejectedExecutionHandler的rejectedExecution方法，线程池有以下四种拒绝策略。
- AbortPolicy：当任务添加到线程池中被拒绝时，它将抛出RejectedExecutionException 异常。
- CallerRunsPolicy：当任务添加到线程池中被拒绝时，会在线程池当前正在运行的Thread线程池中处理被拒绝的任务。
- DiscardOldestPolicy：当任务添加到线程池中被拒绝时，线程池会放弃等待队列中最旧的未处理任务，然后将被拒绝的任务添加到等待队列中。
- DiscardPolicy：当任务添加到线程池中被拒绝时，线程池将丢弃被拒绝的任务。

```
- 构造函数二
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
```
- 构造函数三
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
```
- 构造函数四
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}
```
备注：其中defaultHandler = new AbortPolicy();该拒绝策略是当任务添加到线程池中被拒绝时，它将抛出RejectedExecutionException 异常

### 案例分析
- 案例一：调整核心线程数和最大线程数，分别验证SynchronousQueue，LinkedBlockingDeque，ArrayBlockingQueue 队列
- 案例二：调整核心线程数和最大线程数，分别验证四种RejectedExecutionHandler
- 案例三：设置自定义线程池配置
- [案例代码](https://github.com/vpinfra/sourecode/blob/master/src/main/java/com/vpinfra/juc/ThreadPoolExcutorTest.java)

### 源码分析
- execute 方法
```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //如果线程数小于corePoolSize，则向线程池中添加一个新线程
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //否则将其加入等待执行队列中，如果加入成功，则需要再检查一下是否需要添加新线程或是线程池状态已经发生了变化
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果队列已满，则再尝试添加新的线程
        else if (!addWorker(command, false))
            reject(command);
    }
```
- addWorker 方法
```
/**
* @param firstTask:新增一个线程并执行这个任务，可空，增加的线程从队列获取任务
* @param core:是否使用corePoolSize作为上限，否则使用maxmunPoolSize
**/
private boolean addWorker(Runnable firstTask, boolean core) {
    retry; 
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);// 当前线程池状态
        //RUNNING(-536870912)：运行态，可处理新任务并执行队列中的任务
        //SHUTDOW(0): 待停止态，不接受新任务，但是会处理队列中的任务
        //STOP(536870912):停止态，不接受新任务，不处理队列中任务，且打断运行中任务
        //TIDYING(1073741824): 整理态，所有任务已经结束，workerCount = 0 ，将执行terminated()方法
        //TERMINATED(1610612736): 结束态，terminated() 方法已完成

        if (rs >= SHUTDOWN && !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))// 原子操作递增workCount
                break retry;// 操作成功跳出的重试的循环
            c = ctl.get();
            if (runStateOf(c) != rs)// 如果线程池的状态发生变化则重试
                continue retry;
        }
    }

    // wokerCount递增成功
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock;
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            mainLock.lock();
            try {
                int c = ctl.get();
                int rs = runStateOf(c);

                // RUNNING状态 || SHUTDONW状态下清理队列中剩余的任务
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    // 将新启动的线程添加到线程池中
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动新添加的线程，这个线程首先执行firstTask，然后不停的从队列中取任务执行
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (!workerStarted)
            addWorkerFailed(w);
    }
```
- runWorker 执行任务方法
```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
- getTask 获取队列任务方法
```
private Runnable getTask() {
    boolean timedOut = false;

    retry: 
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
    
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
    
        // 标记从队列中取任务时是否设置超时时间
        boolean timed;
    
        // 1.RUNING状态
        // 2.SHUTDOWN状态，但队列中还有任务需要执行
        for (;;) {
            int wc = workerCountOf(c);
            timed = allowCoreThreadTimeOut || wc > corePoolSize;
    
            //workerQueue.poll超时时timeOut才为true
            if (wc <= maximumPoolSize && !(timedOut && timed))
                break;
    
            if (compareAndDecrementWorkerCount(c))
                return null;
            c = ctl.get();
            if (runStateOf(c) != rs)
                continue retry;
        }
    
        try {
            //以指定的超时时间从队列中取任务,core thread没有超时
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
- processWorkerExit 清理线程池方法
```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly)
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 累加线程池的completedTasks
        completedTaskCount += w.completedTasks;
        // 从线程池中移除超时或者出现异常的线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 尝试停止线程池
    tryTerminate();

    int c = ctl.get();
    // runState为RUNNING或SHUTDOWN
    if (runStateLessThan(c, STOP)) {
        // 线程不是异常结束
        if (!completedAbruptly) {
            // 线程池最小空闲数，允许core thread超时就是0，否则就是corePoolSize
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 如果min == 0但是队列不为空要保证有1个线程来执行队列中的任务
            if (min == 0 && !workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return;
        }
        addWorker(null, false);
    }
}
```
- tryTerminate 终止线程方法
```
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 线程池还处于RUNNING状态、SHUTDOWN状态但是任务队列非空、runState >= TIDYING 线程池已经停止或正在停止时直接返回
        if (isRunning(c) || runStateAtLeast(c, TIDYING) || (runStateOf(c) == SHUTDOWN && !workQueue.isEmpty()))
            return;

        if (workerCountOf(c) != 0) {
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 进入TIDYING状态
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}
```
- shutdown 将runState置为SHUTDOWN，并终止所有空闲的线程
```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 线程池状态设为SHUTDOWN，如果已经至少是这个状态那么则直接返回
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```
- interruptIdleWorkers 终止空闲线程
```
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // w.tryLock能获取到锁，说明该线程没有在运行，因为runWorker中执行任务会先lock，
            // 因此保证了中断的肯定是空闲的线程。
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    }
    finally {
        mainLock.unlock();
    }
}
```
- shutdownNow 将runState置为STOP，并终止所有的线程
```
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // STOP状态：不再接受新任务且不再执行队列中的任务。
        advanceRunState(STOP);
        // 中断所有线程
        interruptWorkers();
        // 返回队列中还没有被执行的任务。
        tasks = drainQueue();
    }
    finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
- interruptWorkers 终止所有的线程，调用的线程的interruptIfStarted方法
```
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```



````