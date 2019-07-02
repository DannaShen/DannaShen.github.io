---
layout: post
title: Java中的线程池之ThreadPoolExecutor及源码
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. 线程池的实现原理
当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？本节来看一下线程池的主要处理流程，处理流程图如图所示。  
<br/>
![](/images/posts/多线程/线程池/ThreadPoolExecutor-execute流程图.png)  
<br/>
从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。  
1）如果当前运行的线程少于核心线程数，则创建一个新的工作线程来执行任务。  
2）如果当前运行的线程等于或多于核心线程数线程池，判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。
如果工作队列满了，则进入下个流程。    
3）线程池(maximumPoolSize中除了核心下线程的那部分见下图)判断线程池的线程是否都处于工作状态。如果没有，
则创建一个新的工作线程来执行任务。否则，则交给饱和策略来处理这个任务。  
ThreadPoolExecutor执行execute()方法的示意图，如图所示。  
<br/>
![](/images/posts/多线程/线程池/ThreadPoolExecutor-execute示意图.png)  
<br/>
ThreadPoolExecutor执行execute方法分下面4种情况。  
1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（执行这一步骤需要获取全局锁）。  
2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。  
3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（执行这一步骤需要获取全局锁）。  
4）如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。  
<br/>   
ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能地避免获取全局锁
（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），
几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。  
## 2. 源码分析
#### 2.1 整体结构
##### 2.1.1 继承关系
![](/images/posts/多线程/线程池/ThreadPoolExecutor-类图.png)  
##### 2.1.2 Executor接口
``` java
public interface Executor {
    void execute(Runnable command);
}
```
##### 2.1.3 ExecutorService接口
``` java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
                                    throws InterruptedException;

    
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
                    throws InterruptedException, ExecutionException;

   
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
                    throws InterruptedException, ExecutionException, TimeoutException;
}
```
ExecutorService接口继承Executor接口，并增加了submit、shutdown、invokeAll等等一系列方法。
##### 2.1.4 AbstractExecutorService抽象类
``` java
public abstract class AbstractExecutorService implements ExecutorService {

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

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

    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos)
                              throws InterruptedException, ExecutionException, TimeoutException {...}

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
                            throws InterruptedException, ExecutionException {... }

    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
                            throws InterruptedException, ExecutionException, TimeoutException {...}

    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
                                         throws InterruptedException {...}

    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
                                        throws InterruptedException {...}

}
```
AbstractExecutorService抽象类实现ExecutorService接口，并且提供了一些方法的默认实现，例如submit方法、invokeAny方法、invokeAll方法。  
<br/>
像execute方法、线程池的关闭方法（shutdown、shutdownNow等等）就没有提供默认的实现。
#### 2.2 构造函数与线程池状态
##### 2.2.1 构造函数
``` java
public ThreadPoolExecutor(int corePoolSize,                             //核心线程数
                              int maximumPoolSize,                      //最大线程数
                              long keepAliveTime,                       //线程存活时间
                              TimeUnit unit,                            //keepAliveTime的单位
                              BlockingQueue<Runnable> workQueue,        //阻塞任务队列
                              ThreadFactory threadFactory,              //创建线程工厂
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
##### 2.2.2 线程池状态
int 是4个字节，也就是32位（注：一个字节等于8位）  
``` java
//记录线程池状态和线程数量（总共32位，前三位表示线程池状态，后29位表示线程数量）
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//线程数量统计位数29  Integer.SIZE=32 
private static final int COUNT_BITS = Integer.SIZE - 3;
//容量 000 11111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

//运行中 111 00000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
//关闭 000 00000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//停止 001 00000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
//整理 010 00000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
//终止 011 00000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

//获取运行状态（获取前3位）
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取线程个数（获取后29位）
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
RUNNING：接受新任务并且处理阻塞队列里的任务  
SHUTDOWN：拒绝新任务但是处理阻塞队列里的任务  
STOP：拒绝新任务并且抛弃阻塞队列里的任务同时会中断正在处理的任务  
TIDYING：所有任务都执行完（包含阻塞队列里面任务），当前线程池活动线程为0，将要调用terminated方法  
TERMINATED：终止状态。terminated方法调用完成以后的状态  
<br/>
线程池状态转换：  
RUNNING -> SHUTDOWN：显式调用shutdown()方法, 或者隐式调用了finalize()方法  
(RUNNING or SHUTDOWN) -> STOP：显式调用shutdownNow()方法  
SHUTDOWN -> TIDYING：当线程池和任务队列都为空的时候  
STOP -> TIDYING：当线程池为空的时候  
TIDYING -> TERMINATED：当 terminated() hook 方法执行完成时候  

<br/>
我们知道线程池的提交模式是submit方法和execute方法。下面看这些方法的实现原理和区别。  

#### 2.2 submit
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
流程步骤如下  

- 调用submit方法，传入Runnable或者Callable对象  
- 判断传入的对象是否为null，为null则抛出异常，不为null继续流程  
- 将传入的对象转换为RunnableFuture对象  
- 执行execute方法，传入RunnableFuture对象  
- 返回RunnableFuture对象  


#### 2.3 execute方法
execute方法在Executor中定义，入参为Runnable接口，没有返回值，线程池没有计算的结果返回。  
``` java
public interface Executor {
   void execute(Runnable command);
}
```
ThreadPoolExecutor中的execute方法的实现：
``` java
    public void execute(Runnable command) {
        //传进来的线程为null，则抛出空指针异常
        if (command == null)
            throw new NullPointerException();
        //获取当前线程池的状态+线程个数变量
        int c = ctl.get();
        /**
        * 3个步骤
        */
        //1.判断当前线程池线程个数是否小于corePoolSize,小于则调用addWorker方法创建新线程运行,
        //且传进来的Runnable当做第一个任务执行。
        //如果调用addWorker方法返回false，则直接返回
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
           2.SHUTDOWN值为0，即如果c小于0，表示在运行；offer用来判断任务是否成功入队*/
        if (isRunning(c) && workQueue.offer(command)) {
             //二次检查
            int recheck = ctl.get();
            //如果当前线程池状态不是RUNNING则从队列删除任务，并执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                //执行拒绝策略
                reject(command);
            //否则如果当前线程池线程空，则添加一个线程
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
```
总结：有三步  
1）检查core线程池数量<corePoolSize数量，是，可以提交任务或新建线程执行任务 。  

2）如果corePoolSize线程数量已使用，接着判断如果队列容量未满，则加入队列。  

3）队列已满，创建maximumPoolSize容量线程执行；如果失败则执行关闭线程池或者拒绝策略。  
![](/images/posts/多线程/线程池/ThreadPoolExecutor之execute.png)  
整个流程的详细步骤  

1. 调用execute方法，传入Runable对象  
2. 判断传入的对象是否为null，为null则抛出异常，不为null继续流程  
3. 获取当前线程池的状态和线程个数变量  
4. 判断当前线程数是否小于核心线程数，是走流程5，否则走流程6  
5. 添加线程数，添加成功则结束，失败则重新获取当前线程池的状态和线程个数变量,  
6. 判断线程池是否处于RUNNING状态，是则添加任务到阻塞队列，否则走流程10，添加任务成功则继续流程7  
7. 重新获取当前线程池的状态和线程个数变量  
8. 重新检查线程池状态，不是运行状态则移除之前添加的任务，有一个false走流程9，都为true则走流程11  
9. 检查线程池线程数量是否为0，否则结束流程，是调用addWorker(null, false)，然后结束  
10. 调用!addWorker(command, false)，为true走流程11，false则结束  
11. 调用拒绝策略reject(command)，结束  
##### 2.3.1 addWorker
``` java
private boolean addWorker(Runnable firstTask, boolean core) {
        //自旋检查
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检查当前线程池状态是否是SHUTDOWN、STOP、TIDYING或者TERMINATED
            // 且！（当前状态为SHUTDOWN、且传入的任务为null，且队列不为null）
            // 条件都成立则返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                 //如果当前的线程数量超过最大容量或者大于（根据传入的core决定是核心线程数还是最大线程数）
                 //核心线程数 || 最大线程数，则返回false
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //线程数+1，CAS原子操作，成功后跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //CAS失败执行下面方法，查看当前线程数是否变化，变化则继续retry循环，没变化则继续内部循环
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }
        //CAS成功
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
                    //重新检查线程池状态
                    //避免ThreadFactory退出故障或者在锁获取前线程池被关闭
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
                //判断worker是否添加成功，成功则启动线程，然后将workerStarted设置为true
                if (workerAdded) {
                    //上面的标记，启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //判断线程有没有启动成功，没有则调用addWorkerFailed方法
            if (! workerStarted)
                //失败处理
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
这里可以将addWorker分为两部分，第一部分增加线程池个数，第二部分是将任务添加到workder里面并执行。  
<br/>
第一部分主要是两个循环，外层循环主要是判断线程池状态：  
``` java
rs >= SHUTDOWN &&
              ! (rs == SHUTDOWN &&
                  firstTask == null &&
                  ! workQueue.isEmpty())
```
展开！运算后等价于  
``` java
rs >= SHUTDOWN &&
                (rs != SHUTDOWN ||
              firstTask != null ||
              workQueue.isEmpty())
```
也就是说下面几种情况下会返回false：  

- 当前线程池状态为STOP，TIDYING，TERMINATED  
- 当前线程池状态为SHUTDOWN并且已经有了第一个任务  
- 当前线程池状态为SHUTDOWN且没有第一个任务且任务队列为空  

<br/>
内层循环作用是使用cas增加线程个数，如果线程个数超限则返回false，否者进行cas，cas成功则退出双循环，否者cas失败了，
要看当前线程池的状态是否变化了，如果变了，则重新进入外层循环重新获取线程池状态，否者进入内层循环继续进行cas尝试。  
<br/>
到了第二部分说明CAS成功了，也就是说线程个数加一了，但是现在任务还没开始执行，这里使用全局的独占锁来控制workers里面添加任务，
其实也可以使用并发安全的set，但是性能没有独占锁好（这个从注释中知道的）。这里需要注意的是要在获取锁后重新检查线程池的状态，
这是因为其他线程可可能在本方法获取锁前改变了线程池的状态，比如调用了shutdown方法。添加成功则启动任务执行。  
<br/>

第一部分流程图  
![](/images/posts/多线程/线程池/ThreadPoolExecutor-addWorker第1部分.png)  
第二部分流程图  
![](/images/posts/多线程/线程池/ThreadPoolExecutor-addWorker第2部分.png)  

##### 2.3.2 Worker
Worker是定义在ThreadPoolExecutor中的finnal类，其中继承了AbstractQueuedSynchronizer类和实现Runnable接口，其中的run方法如下

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
线程启动时调用了runWorker方法
##### 2.3.3 runWorker
``` java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //任务线程的锁状态默认为-1（构造函数设置的），此时解锁+1，变为0，是锁打开状态，允许中断。  
        w.unlock(); 
        boolean completedAbruptly = true;
        try {
            //循环获取任务，判断添加的任务存在或者队列中的任务不为空
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 判断线程池是处于STOP状态或者TIDYING、TERMINATED状态，当不是这几个状态，也就是
                // 当前线程就处于RUNNING或者SHUTDOWN状态时，设置当前线程处于中断状态
                // 接着判断当前线程是否中断，如果未中断，则进行中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) 
                        && !wt.isInterrupted())
                    //中断当前线程
                    wt.interrupt();
                try {
                    //提供给继承类使用做一些统计之类的事情，在线程运行前调用
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
                        //提供给继承类使用做一些统计之类的事情，在线程运行之后调用
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                     //统计当前worker完成了多少个任务
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //整个线程结束时调用，线程退出操作。统计整个线程池完成的任务个数之类的工作
            processWorkerExit(w, completedAbruptly);
        }
    }
```
##### 2.3.4 getTask
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

            //工作线程数超过max，或者队列为空，工作线程存在
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
##### 2.3.5 processWorkerExit
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
##### 2.3.6 addWorkerFailed(w);
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
##### 2.3.7 tryTerminate
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
        }
    }
```
##### 2.3.8 interruptIdleWorkers
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
#### 2.4 关闭线程池
##### 2.4.1 原理
可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，
然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，
shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而
shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。  
<br/>
只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，
这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，
通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。
##### 2.4.2 shutdown
当调用shutdown方法时，线程池将不会再接收新的任务，然后将先前放在队列中的任务执行完成。
``` java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查权限
            checkShutdownAccess();
            //CAS 更新线程池状态
            advanceRunState(SHUTDOWN);
            //中断所有空闲的线程
            interruptIdleWorkers();
            //关闭，此处是do nothing
            onShutdown();
        } finally {
            mainLock.unlock();
        }
        //尝试结束，上面代码已分析
        tryTerminate();
    }
```
##### 2.4.3 shutdownNow
立即停止所有的执行任务，并将队列中的任务返回
``` java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        //中断所有线程
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
##### 2.4.4 shutdown和shutdownNow区别
shutdown和shutdownNow这两个方法的作用都是关闭线程池，流程大致相同，只有几个步骤不同，如下  
1. 加锁  
2. 检查关闭权限  
3. CAS改变线程池状态  
4. 设置状态  
5. 中断空闲的线程/中断所有线程  
6. onShutdown(do nothing)/获取队列中的任务
7. 解锁  
8. 尝试将线程池状态变成终止状态TERMINATED  
8. 结束/返回队列中的任务  

#### 2.5 总结
1）线程池优先使用corePoolSize的数量执行工作任务  

2）如果超过corePoolSize，队列入队  

3）超过队列，使用maximumPoolSize-corePoolSize的线程处理，这部分线程超时不干活就销毁掉。  

4）每个线程执行结束的时候，会判断当前的工作线程和任务数，如果任务数多，就会创建空线程从队列拿任务。  

5）线程池执行完成，不会自动销毁，需要手工shutdown，修改线程池状态，中断所有线程。  
## 3. 合理配置线程池
要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析。  

- 任务的性质：CPU密集型任务、IO密集型任务和混合型任务。  
- 任务的优先级：高、中和低。  
- 任务的执行时间：长、中和短。  
- 任务的依赖性：是否依赖其他系统资源，如数据库连接。  

<br/>

性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务应配置尽可能小的线程，如配置cpu个数 +1个线程的线程池。
由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2*cpu个数 。混合型的任务，如果可以拆分，
将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量
将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。可以通过
Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。
优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先执行。  
<br/>
执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让执行时间短的任务先执行。  
<br/>
依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，
这样才能更好地利用CPU。  
#### 3.1 建议使用有界队列
有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。有一次，我们系统里后台任务线程池的队列和线程池全满了，
不断抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，
因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞，任务积压在线程池里。
如果当时我们设置成无界队列，那么线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。
当然，我们的系统所有的任务是用单独的服务器部署的，我们使用不同规模的线程池完成不同类型的任务，但是出现这样问题时也会影响到其他任务。
## 4. 线程池监控
如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。
可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性。  
- taskCount：线程池需要执行的任务数量。  
- completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。  
- largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，
则表示线程池曾经满过。  
- getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。  
- getActiveCount：获取活动的线程数。  
<br/>
通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，
也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。例如，监控任务的平均执行时间、最大执行时间和最小执行时间等。
这几个方法在线程池里是空方法。
