---
layout: post
title: Java中线程池
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. 线程池的实现原理
当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？本节来看一下线程池的主要处理流程，处理流程图如图所示。  
![](/images/posts/多线程/)  
从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。  
1）线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。
如果核心线程池里的线程都在执行任务，则进入下个流程。  
2）线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。  
3）线程池(maximumPoolSize中除了核心下线程的那部分见下图)判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。否则，则交给饱和策略来处理这个任务。
ThreadPoolExecutor执行execute()方法的示意图，如图所示。  
![](/images/posts/多线程/)  
ThreadPoolExecutor执行execute方法分下面4种情况。  
1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）。  
2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。  
3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。  
4）如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。  
<br/>
ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能地避免获取全局锁
（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），
几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。  
## 2. 源码分析
#### 2.1 构造函数
``` java
public ThreadPoolExecutor(int corePoolSize,                             //核心池大小
                              int maximumPoolSize,                      //最大线程池大小
                              long keepAliveTime,                       //线程存活时间
                              TimeUnit unit,                            //keepAliveTime的单位
                              BlockingQueue<Runnable> workQueue,        //阻塞任务队列
                              ThreadFactory threadFactory,              //线程工厂
                              RejectedExecutionHandler handler)         //拒绝任务的接口处理器
     { 
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
ThreadPoolExecutor继承关系
![](/images/posts/多线程/)  
我们知道线程池的提交模式是submit方法和execute方法。下面看这些方法的实现原理和区别。  
#### 2.2 execute方法
execute方法在Executor中定义，入参为Runnable接口，没有返回值，线程池没有计算的结果返回。  
``` java
public interface Executor {
   void execute(Runnable command);
}
```
ThreadPoolExecutor中的execute方法的实现：
``` java
    /*  ctlOf方法定义：
          private static int ctlOf(int rs, int wc)
          { return rs | wc; }
        即是Runing代表的值和0的或操作，所以结果就是Runing代表的值*/
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //线程池状态码
    private static final int COUNT_BITS = Integer.SIZE - 3;     //29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    //状态码，即 -1，0，1，2，3 左移29位
    //running的状态小于0，每次添加一个线程，做++操作。
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        //AtomicInteger存
        int c = ctl.get();
        //工作线程(运行中的线程)小于corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            //添加一个core线程(核心线程)。此处参数的true，表示添加的线程是core容量下的线程
            if (addWorker(command, true))
                return;
            //刷新数据，乐观锁就是没有锁
            c = ctl.get();
        }
       /*  isRunning方法的定义：
               private static boolean isRunning(int c)
               {return c < SHUTDOWN;}
           SHUTDOWN值为0，即如果c小于0，表示在运行；offer用来判断任务是否成功入队*/
        if (isRunning(c) && workQueue.offer(command)) {
             //再次获取RUNNING值
            int recheck = ctl.get();
            //线程池已停止，就移除队列
            if (! isRunning(recheck) && remove(command))
                //执行拒绝策略
                reject(command);
            //线程池在运行，有效线程数为0 
            else if (workerCountOf(recheck) == 0)
                //添加一个空线程进线程池，使用非core容量线程
                //仅有一种情况，会走这步，core线程数为0，max线程数>0,队列容量>0
                //创建一个非core容量的线程，线程池会将队列的command执行
                addWorker(null, false);
        }
        //线程池停止了或者队列已满，添加maximumPoolSize容量工作线程，如果失败，执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }

    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```
总结：有三步  
1）检查core线程池数量<corePoolSize数量，是，可以提交任务或新建线程执行任务 。  

2）如果corePoolSize线程数量已使用，接着判断如果队列容量未满，则加入队列。  

3）队列已满，创建maximumPoolSize容量线程执行；如果失败则执行关闭线程池或者拒绝策略。  
##### 2.2.1 addWorker
``` java
private boolean addWorker(Runnable firstTask, boolean core) {
        //自旋检查
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //如果线程池已关闭；线程池正在关闭，提交的任务为null，任务队列不为空，则直接返回失败  
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //工作线程数达到或超过最大容量，或者添加core线程达到最大容量或者添加max线程达到最大容量，直接失败
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //线程数+1，CAS原子操作，成功后跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //原子操作失败，判断线程池运行状态是否已改变
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //所有代码的核心，new Worker，创建了线程，或者复用线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                //线程池是操作多线程，加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                   
                    int rs = runStateOf(ctl.get());

                    //再检查一次，线程池在运行或正在停止
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //添加的线程不能是活跃线程
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        //添加线程存储，HashSet存储
                        workers.add(w);
                        //重入锁最大池数量
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //上面的标记，启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                //失败处理
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
##### 2.2.2 Worker
``` java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
{
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;
    Worker(Runnable firstTask) {
        setState(-1); 
        this.firstTask = firstTask;
        //此处创建了线程，线程工厂使用this
        this.thread = getThreadFactory().newThread(this);
    }
    public void run() {
         runWorker(this);
    }

}
```
Worker实现AQS和Runnable接口 ，是线程接口实现
##### 2.2.3 run
``` java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //任务线程的锁状态默认为-1（构造函数设置的），此时解锁+1，变为0，是锁打开状态，允许中断。  
        w.unlock(); 
        boolean completedAbruptly = true;
        try {
            //如果添加的任务存在或者队列中的任务不为空
            while (task != null || (task = getTask()) != null) {
                w.lock();
                
                //线程池正在停止，当前线程未中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    //中断当前线程
                    wt.interrupt();
                try {
                    //准备，空方法，可自定义实现
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //本质，直接调用run方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //结束，空方法，可自定义实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    //记录任务数
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //worker结束处理
            processWorkerExit(w, completedAbruptly);
        }
    }
```
##### 2.2.4 getTask
``` java
private Runnable getTask() {
        boolean timedOut = false; 
        //自旋锁
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //检查线程池是否关闭，队列为空
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                //减少工作线程数量
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            //工作线程数超过core或者max，或者队列为空，工作线程存在
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                //减少任务数
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //超时机制控制队列取元素
                //take        移除并返回队列头部的元素     如果队列为空，则阻塞
                //poll        移除并返问队列头部的元素    如果队列为空，则返回null
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

```
##### 2.2.5 processWorkerExit
``` java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        //正常执行，此处设置为法false
        if (completedAbruptly)
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //任务数增加
            completedTaskCount += w.completedTasks;
            //移除HashSet的线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        //尝试停止线程池
        tryTerminate();

        int c = ctl.get();
        //如果没有停止线程池
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                //core线程数量为0或者队列为空，默认1
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                //工作线程比core线程多，直接返回
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //当前线程运行结束，增加空线程，容量maximumPoolSize
            addWorker(null, false);
        }
    }
```
##### 2.2.6 addWorkerFailed(w);
``` java
private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                //工作线程移除任务HashSet
                workers.remove(w);
            //工作线程数量-1
            decrementWorkerCount();
            //尝试停止线程池
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

```
##### 2.2.7 tryTerminate
``` java
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //线程池在运行或者状态在变化中，或者正在停止但队列有任务，直接返回
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            //有任务工作线程
            if (workerCountOf(c) != 0) { 
                //中断所有线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //线程池状态改变
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //结束线程池，空方法，does nothing
                        terminated();
                    } finally {
                        //设置线程池状态，结束
                        ctl.set(ctlOf(TERMINATED, 0));
                        //唤醒所有wait线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
##### 2.2.8 interruptIdleWorkers
``` java
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
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
        } finally {
            mainLock.unlock();
        }
    }
```
#### 2.3 shutdown
``` java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查权限
            checkShutdownAccess();
            //CAS 更新线程池状态
            advanceRunState(SHUTDOWN);
            //中断所有线程
            interruptIdleWorkers();
            //关闭，此处是do nothing
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        //尝试结束，上面代码已分析
        tryTerminate();
    }
```
#### 2.4 submit
``` java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```
调用的execute方法，只是多了newTaskFor，用来收集线程的运算结果。
#### 2.5 总结
1）线程池优先使用corePoolSize的数量执行工作任务  

2）如果超过corePoolSize，队列入队  

3）超过队列，使用maximumPoolSize-corePoolSize的线程处理，这部分线程超时不干活就销毁掉。  

4）每个线程执行结束的时候，会判断当前的工作线程和任务数，如果任务数多，就会创建空线程从队列拿任务。  

5）线程池执行完成，不会自动销毁，需要手工shutdown，修改线程池状态，中断所有线程。  

