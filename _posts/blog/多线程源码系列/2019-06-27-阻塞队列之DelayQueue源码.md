---
layout: post
title: Java中的阻塞队列之DelayQueue源码
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. DelayQueue介绍
DelayQueue 是一个支持延时获取元素的无界阻塞队列，队列使用PriorityQueue来实现，队列中的元素必须实现Delayed接口，
在创建元素时可以指定多久才能从队列中获取当前元素，只有在延时期满时才能从队列中提取元素。  
<br/>
PriorityQueue 是一种优先级的队列，队列中的元素会按照优先级进行排序，PriorityQueue其实现原理也是使用的二叉堆。  
## 2. 接口分析
``` java
public interface Delayed extends Comparable<Delayed> {

    long getDelay(TimeUnit unit);
}
```
Delayed 接口有一个getDelay 方法接口，该方法用来告知延迟到期有多长的时间，或者延迟在多长时间之前已经到期。  
<br/>
为了排序Delayed 接口还继承了Comparable 接口，因此必须实现compareTo()，使其可以进行元素的比较。  
## 3. 继承体系
![](/images/posts/多线程/源码系列/)  
和阻塞队列ArrayBlockingQueue，LinkedBlockingQueue，PriorityBlockingQueue基本是一样的，唯独没有实现序列化接口。  
DelayQueue 实现了BlockingQueue接口，该接口中定义了阻塞的方法接口。  
DelayQueue 继承了AbstractQueue，具有了队列的行为。  
## 4. DelayQueue数据结构
``` java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E> implements BlockingQueue<E> {
    //可重入锁
    private final transient ReentrantLock lock = new ReentrantLock();
    //存储元素的优先级队列
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    //获取数据 等待线程标识
    private Thread leader = null;

    //条件控制，表示是否可以从队列中取数据
    private final Condition available = lock.newCondition();
}
```
DelayQueue 通过组合一个PriorityQueue 来实现元素的存储以及优先级维护，通过ReentrantLock 来保证线程安全，
通过Condition 来判断是否可以取数据，对于leader我们后面再来分析它的作用。  
## 5. 构造方法
#### 5.1 默认构造方法
``` java
    public DelayQueue() {}
```
#### 5.2 通过集合初始化
``` java
    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
```
DelayQueue 内部组合PriorityQueue，对元素的操作都是通过PriorityQueue 来实现的，DelayQueue 的构造方法很简单，
对于PriorityQueue 都是使用的默认参数，不能通过DelayQueue 来指定PriorityQueue的初始大小，也不能使用指定的Comparator，
元素本身就需要实现Comparable ，因此不需要指定的Comparator。
## 6. 入队
#### 6.1 add(E e)
将指定的元素插入到此队列中，在成功时返回 true
``` java
    public boolean add(E e) {
        return offer(e);
    }
```
#### 6.2 offer(E e)
将指定的元素插入到此队列中，在成功时返回 true，在前面的add 中，内部调用了offer 方法，我们也可以直接调用offer 方法来完成入队操作。  
``` java
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        //获取锁
        lock.lock();
        try {
            //将元素存入PriorityQueue，队列中会按照优先级排序，刚刚放入的元素并不一定就在队列的第一个位置
            q.offer(e);
            //q.peek()获取的是队列的第一个元素
            //如果刚刚放进去的元素就在第一个位置，唤醒正在等待的线程
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```
peek并不一定是当前添加的元素，队头是当前添加元素，说明当前元素e的优先级最小也就即将过期的，
这时候激活avaliable变量条件队列里面的一个线程，通知他们队列里面有元素了。
#### 6.3 offer(E e, long timeout, TimeUnit unit)
``` java
public boolean offer(E e, long timeout, TimeUnit unit) {
        //调用offer 方法
        return offer(e);
    }
```
因为是无界队列，因此不会出现”队满”(超出最大值会抛异常)，指定一个等待时间将元素放入队列中并没有意义，
队列没有达到最大值那么会入队成功，达到最大值，则失败，不会进行等待。  
#### 6.4 put(E e)
将指定的元素插入此队列中,队列达到最大值，则抛oom异常
``` java
    public void put(E e) {
        offer(e);
    }
```
虽然提供入队的接口方式很多，实际都是调用的offer 方法，通过PriorityQueue 来进行入队操作，入队超时方法并没有其超时功能。
## 7. 出队
#### 7.1 poll()
获取并移除此队列的头，如果此队列为空，则返回 null  
``` java
public E poll() {
    final ReentrantLock lock = this.lock;
    //获取同步锁
    lock.lock();
    try {
        //获取队头
        E first = q.peek();
        //如果队头为null 或者 延时还没有到，则返回null
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll(); //元素出队
    } finally {
        lock.unlock();
    }
}
```
#### 7.2 poll(long timeout, TimeUnit unit)
获取并移除此队列的头部，在指定的等待时间前等待。  
``` java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    //超时等待时间
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    //可中断的获取锁
    lock.lockInterruptibly();
    try {
        //无限循环
        for (;;) {
            //获取队头元素
            E first = q.peek();
            //队头为空，也就是队列为空
            if (first == null) {
                //达到超时指定时间，返回null 
                if (nanos <= 0)
                    return null;
                else
                    // 如果还没有超时，那么再available条件上进行等待nanos时间
                    nanos = available.awaitNanos(nanos);
            } else {
                //获取元素延迟时间
                long delay = first.getDelay(NANOSECONDS);
                //延时到期
                if (delay <= 0)
                    return q.poll(); //返回出队元素
                //延时未到期，超时到期，返回null
                if (nanos <= 0)
                    return null;
                first = null; // don't retain ref while waiting
                // 超时等待时间 < 延迟时间 或者有其它线程再取数据
                if (nanos < delay || leader != null)
                    //在available 条件上进行等待nanos 时间
                    nanos = available.awaitNanos(nanos);
                else {
                    //超时等待时间 > 延迟时间 并且没有其它线程在等待，
                    //那么当前线程成为leader，表示leader 线程最早 处于等待获取元素
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                    //等待  延迟时间 超时
                        long timeLeft = available.awaitNanos(delay);
                        //还需要继续等待 nanos
                        nanos -= delay - timeLeft;
                    } finally {
                        //清除 leader
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //唤醒阻塞在available 的一个线程，表示可以取数据了
        if (leader == null && q.peek() != null)
            available.signal();
        //释放锁
        lock.unlock();
    }
}
```
总结：  
1. 如果队列为空，且超时时间未到，则进行等待，否则返回null  
2. 队列不空，如果延迟时间到，则取出队头元素并返回，否则(延迟时间未到)且超时时间到 返回null  
3. 队列不空，如果延迟时间到，则取出队头元素并返回，否则(延迟时间未到)且**超时时间未到**，
并且 超时时间< 延迟时间或者有线程正在获取元素，那么进行等待  
4. 队列不空，如果延迟时间到，则取出队头元素并返回，否则(延迟时间未到)且**超时时间未到**，
并且 超时时间> 延迟时间，那么肯定可以取到元素，设置leader为当前线程，等待延迟时间到期。  

这里需要注意的时Condition 条件在阻塞时会释放锁，在被唤醒时会再次获取锁，获取成功才会返回。  
当进行超时等待时，阻塞在Condition 上后会释放锁,一旦释放了锁，那么其它线程就有可能参与竞争，
某一个线程就可能会成为leader(参与竞争的时间早，并且能在等待时间内能获取到队头元素那么就可能成为leader)
leader是用来减少不必要的竞争,如果leader不为空说明已经有线程在取了,设置当前线程等待即可。
（leader 就是一个信号，告诉其它线程：你们不要再去获取元素了，它们延迟时间还没到期，我都还没有取到数据呢，你们要取数据，
等我取了再说）  
<br/>
下面用流程图来展示这一过程：  
#### 7.3 take()
获取并移除此队列的头部，在元素变得可用之前一直等待  
``` java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    //延迟到期
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    //如果有其它线程在等待获取元素，则当前线程不用去竞争，直接等待
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            //等待延迟时间到期
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```
该方法就是相当于在前面的超时等待中，把超时时间设置为无限大，那么这样只要队列中有元素，如果元素延迟时间到期，
那么就可以取出元素，否则就直接等待元素延迟时间到期，再取出元素，最先参与等待的线程会成为leader。
#### 7.4 peek()
调用此方法，可以返回队头元素，但是元素并不出队。  
``` java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //返回队列头部元素，元素不出队
            return q.peek();
        } finally {
            lock.unlock();
        }
    }
```
## 8. 总结：  

- DelayQueue 内部通过组合PriorityQueue 来实现存储和维护元素顺序的。  
- DelayQueue 存储元素必须实现Delayed 接口，通过实现Delayed 接口，可以获取到元素延迟时间，
以及可以比较元素大小（Delayed 继承Comparable）  
- DelayQueue 通过一个可重入锁来控制元素的入队出队行为  
- DelayQueue 中leader 标识 用于减少线程的竞争，表示当前有其它线程正在获取队头元素。  
- PriorityQueue 只是负责存储数据以及维护元素的顺序，对于延迟时间取数据则是在DelayQueue 中进行判断控制的。  
- DelayQueue 没有实现序列化接口  




