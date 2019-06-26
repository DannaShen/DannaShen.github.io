---
layout: post
title: Condition接口实现
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1.概述
Condition接口的await/signal机制是设计用来代替监视器锁(synchronized)的wait/notify机制的，因此，
与监视器锁的wait/notify机制对照着学习有助于我们更好的理解Conditon接口:  

| Object 方法 | Condition 方法 | 区别 |
| :------| :------ | :------ |
| void wait() | void await() |  |
| void wait(long timeout) | long awaitNanos(long nanosTimeout) | 时间单位，返回值 |
| void wait(long timeout, int nanos | 	boolean await(long time, TimeUnit unit) | 时间单位，参数类型，返回值 |
| void notify() | void signal() |  | 
| void notifyAll() | void signalAll() |  | 
| - | void awaitUninterruptibly() | Condition独有 |
| - | boolean awaitUntil(Date deadline) | Condition独有 | 

这里先做一下说明，本文说wait方法时，是泛指wait()、wait(long timeout)、wait(long timeout, int nanos) 三个方法，
当需要指明某个特定的方法时，会带上相应的参数。同样的，说notify方法时，也是泛指notify()，notifyAll()方法，
await方法和signal方法以此类推。  
<br/>
首先，我们通过wait/notify机制来类比await/signal机制：  

- 1. 调用wait方法的线程首先必须是已经进入了同步代码块，即已经获取了监视器锁；与之类似，调用await方法的线程首先必须获得lock锁  
- 2. 调用wait方法的线程会释放已经获得的监视器锁，进入当前监视器锁的等待队列（wait set）中；与之类似，
调用await方法的线程会释放已经获得的lock锁，进入到当前Condtion对应的条件队列中。  
- 3. 调用监视器锁的notify方法会唤醒等待在该监视器锁上的线程，这些线程将开始参与锁竞争，并在获得锁后，从wait方法处恢复执行；
与之类似，调用Condtion的signal方法会唤醒对应的条件队列中的线程，这些线程将开始参与锁竞争，并在获得锁后，从await方法处开始恢复执行。  

<br/>

## 2. 同步队列 vs 条件队列
#### 2.1 sync queue
同步队列如下图：  
![](/images/posts/多线程/源码系列/AQS源码-Condition-同步队列.png)  
sync queue是一个双向链表，我们使用prev、next属性来串联节点。但是在这个同步队列中，我们一直没有用到nextWaiter属性，
即使是在共享锁模式下，这一属性也只作为一个标记，指向了一个空节点，因此，在sync queue中，我们不会用它来串联节点。  
#### 2.2 condition queue
每创建一个Condition对象就会对应一个Condition队列，每一个调用了Condition对象的await方法的线程都会被包装成Node扔进一个条件队列中，
就像这样：
![](/images/posts/多线程/源码系列/AQS源码-Condition-条件队列.png)  
可见，每一个Condition对象对应一个Condition队列，每个Condition队列都是独立的，互相不影响的。在上图中，
如果我们对当前线程调用了notFull.await(), 则当前线程就会被包装成Node加到notFull队列的末尾。  
<br/>
值得注意的是，condition queue是一个单向链表，在该链表中我们使用nextWaiter属性来串联链表。但是，
就像在sync queue中不会使用nextWaiter属性来串联链表一样，在condition queue中，也并不会用到prev, next属性，它们的值都为null。
也就是说，在条件队列中，Node节点真正用到的属性只有三个：  

- thread：代表当前正在等待某个条件的线程  
- waitStatus：条件的等待状态  
- nextWaiter：指向条件队列中的下一个节点  
<br/>
既然这里又提到了waitStatus，我们这里再回顾一下它的取值范围：  
``` java
volatile int waitStatus;
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;
```
在条件队列中，我们只需要关注一个值即可——CONDITION。它表示线程处于正常的等待状态，而只要waitStatus不是CONDITION，
我们就认为线程不再等待了，此时就要从条件队列中出队。  
#### 2.3 sync queue 和 conditon queue的联系
一般情况下，等待锁的sync queue和条件队列condition queue是相互独立的，彼此之间并没有任何关系。但是，
当我们调用某个条件队列的signal方法时，会将某个或所有等待在这个条件队列中的线程唤醒，被唤醒的线程和普通线程一样需要去争锁，
如果没有抢到，则同样要被加到等待锁的sync queue中去，此时节点就从condition queue中被转移到sync queue中。  
<br/>
但是，这里尤其要注意的是，node是被一个一个转移过去的，哪怕我们调用的是signalAll()方法也是一个一个转移过去的，
而不是将整个条件队列接在sync queue的末尾。  
<br/>
同时要注意的是，我们在sync queue中只使用prev、next来串联链表，而不使用nextWaiter;
我们在condition queue中只使用nextWaiter来串联链表，而不使用prev、next.事实上，
它们就是两个使用了同样的Node数据结构的完全独立的两种链表。因此，将节点从condition queue中转移到sync queue中时，
我们需要断开原来的链接（nextWaiter）,建立新的链接（prev, next），这某种程度上也是需要将节点一个一个地转移过去的原因之一。  
#### 2.4 入队时和出队时的锁状态
sync queue是等待锁的队列，当一个线程被包装成Node加到该队列中时，必然是没有获取到锁；当处于该队列中的节点获取到了锁，
它将从该队列中移除(事实上移除操作是将获取到锁的节点设为新的dummy head,并将thread属性置为null)。  
<br/>
condition队列是等待在特定条件下的队列，因为调用await方法时，必然是已经获得了lock锁，
所以在进入condtion队列前线程必然是已经获取了锁；在被包装成Node扔进条件队列中后，线程将释放锁，然后挂起；
当处于该队列中的线程被signal方法唤醒后，由于队列中的节点在之前挂起的时候已经释放了锁，所以必须先去再次的竞争锁，因此，
该节点会被添加到sync queue中。因此，条件队列在出队时，线程并不持有锁。  
<br/>
所以事实上，这两个队列的锁状态正好相反：  

- condition queue：入队时已经持有了锁 -> 在队列中释放锁 -> 离开队列时没有锁 -> 转移到sync queue  
- sync queue：入队时没有锁 -> 在队列中争锁 -> 离开队列时获得了锁  
## 3. CondtionObject
AQS对Condition这个接口的实现主要是通过ConditionObject，上面已经说个，它的核心实现就是是一个条件队列，
每一个在某个condition上等待的线程都会被封装成Node对象扔进这个条件队列。  
#### 3.1 核心属性
``` java
private transient Node firstWaiter;
private transient Node lastWaiter;
```
这两个属性分别代表了条件队列的队头和队尾，每当我们新建一个conditionObject对象，都会对应一个条件队列。  
#### 3.2 构造函数
``` java
public ConditionObject() { }
```
构造函数啥也没干，可见，条件队列是延时初始化的，在真正用到的时候才会初始化。
## 4. Condition接口方法实现
#### 4.1 await()第一部分分析
``` java
public final void await() throws InterruptedException {
    // 如果当前线程在调用await()方法前已经被中断了，则直接抛出InterruptedException
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程封装成Node添加到条件队列
    Node node = addConditionWaiter();
    // 释放当前线程所占用的锁，保存当前的锁状态
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 如果当前队列不在同步队列中，说明刚刚被await, 还没有人调用signal方法，则直接将当前线程挂起
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 线程将在这里被挂起，停止运行
        // 能执行到这里说明要么是signal方法被调用了，要么是线程被中断了
        // 所以检查下线程被唤醒的原因，如果是因为中断被唤醒，则跳出while循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 第一部分就分析到这里，下面的部分我们到第二部分再看, 先把它注释起来
    /*
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    */
}
```
##### 4.1.1 addConditionWaiter
``` java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果尾节点被cancel了，则先遍历整个链表，清除所有被cancel的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 将当前线程包装成Node扔进条件队列
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    /*
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
    */
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```
首先我们要思考的是，存在两个不同的线程同时入队的情况吗？不存在。为什么呢？因为前面说过了，
能调用await方法的线程必然是已经获得了锁，而获得了锁的线程只有一个，所以这里不存在并发，因此不需要CAS操作。  
<br/>
在这个方法中，我们就是简单的将当前线程封装成Node加到条件队列的末尾。这和将一个线程封装成Node加入等待队列略有不同：  

- 1. 节点加入sync queue时waitStatus的值为0，但节点加入condition queue时waitStatus的值为Node.CONDTION。  
- 2. sync queue的头节点为dummy节点，如果队列为空，则会先创建一个dummy节点，再创建一个代表当前线程的Node添加在dummy节点的后面；
而condtion queue 没有dummy节点，初始化时，直接将firstWaiter和lastWaiter直接指向新建的节点就行了。  
- 3. sync queue是一个双向队列，在节点入队后，要同时修改当前节点的前驱和前驱节点的后继；而在condtion queue中，
我们只修改了前驱节点的nextWaiter,也就是说，condtion queue是作为单向队列来使用的。  

<br/>

如果入队时发现尾节点已经取消等待了，那么我们就不应该接在它的后面，此时需要调用unlinkCancelledWaiters来剔除那些已经取消等待的线程

###### 4.1.1.1 unlinkCancelledWaiters
``` java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```
该方法将从头节点开始遍历整个队列，剔除其中waitStatus不为Node.CONDTION的节点，
这里使用了两个指针firstWaiter和trail来分别记录第一个和最后一个waitStatus不为Node.CONDTION的节点，这些都是基础的链表操作了。
##### 4.1.2 fullyRelease
在节点被成功添加到队列的末尾后，我们将调用fullyRelease来释放当前线程所占用的锁：
``` java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```
首先，当我们调用这个方法时，说明当前线程已经被封装成Node扔进条件队列了。在该方法中，我们通过release方法释放锁。  
<br/>
值得注意的是，这是一次性释放了所有的锁，即对于可重入锁而言，无论重入了几次，这里是一次性释放完的，
这也就是为什么该方法的名字叫fullyRelease。但这里尤其要注意的是release(savedState)方法是有可能抛出IllegalMonitorStateException的，
这是因为当前线程可能并不是持有锁的线程。但是咱前面不是说，只有持有锁的线程才能调用await方法吗？
既然fullyRelease方法在await方法中，为啥当前线程还有可能并不是持有锁的线程呢？  
<br/>
虽然话是这么说，但是在调用await方法时，我们其实并没有检测Thread.currentThread() == getExclusiveOwnerThread()，换句话说，
也就是执行到fullyRelease这一步，我们才会检测这一点，而这一点检测是由AQS子类实现tryRelease方法来保证的，例如，
ReentrantLock对tryRelease方法的实现如下：
``` java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
当发现当前线程不是持有锁的线程时，我们就会进入finally块，将当前Node的状态设为Node.CANCELLED，
这也就是为什么上面的addConditionWaiter在添加新节点前每次都会检查尾节点是否已经被取消了。  
<br/>
在当前线程的锁被完全释放了之后，我们就可以调用LockSupport.park(this)把当前线程挂起，等待被signal了。但是，
在挂起当前线程之前我们先用isOnSyncQueue确保了它不在sync queue中，这是为什么呢？
当前线程不是在一个和sync queue无关的条件队列中吗？怎么可能会出现在sync queue中的情况？  
##### 4.1.3 isOnSyncQueue
``` java
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    
    return findNodeFromTail(node);
}
```
###### 4.1.3.1 findNodeFromTail
``` java
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```
为了解释这一问题，我们先来看看signal方法
#### 4.2 signalAll()
在看signalAll之前，我们首先要区分调用signalAll方法的线程与signalAll方法要唤醒的线程（等待在对应的条件队列里的线程）：  

- 调用signalAll方法的线程本身是已经持有了锁，现在准备释放锁了；  
- 在条件队列里的线程是已经在对应的条件上挂起了，等待着被signal唤醒，然后去争锁。  

<br/>

首先，与调用notify时线程必须是已经持有了监视器锁类似，在调用condition的signal方法时，线程也必须是已经持有了lock锁：  
``` java
private boolean findNodeFromTail(Node node) {
    public final void signalAll() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignalAll(first);
    }
}
```
该方法首先检查当前调用signal方法的线程是不是持有锁的线程，这是通过isHeldExclusively方法来实现的，该方法由继承AQS的子类来实现，
例如，ReentrantLock对该方法的实现为：
``` java
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```
因为exclusiveOwnerThread保存了当前持有锁的线程，这里只要检测它是不是等于当前线程就行了。
接下来先通过firstWaiter是否为空判断条件队列是否为空，如果条件队列不为空，则调用doSignalAll方法：
##### 4.2.1 doSignalAll
``` java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
首先我们通过lastWaiter = firstWaiter = null;将整个条件队列清空，然后通过一个do-while循环，
将原先的条件队列里面的节点一个一个拿出来(令nextWaiter = null)，再通过transferForSignal方法一个一个添加到sync queue的末尾：
##### 4.2.1.1 transferForSignal

``` java
final boolean transferForSignal(Node node) {
    // 如果该节点在调用signal方法前已经被取消了，则直接跳过这个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 如果该节点在条件队列中正常等待，则利用enq方法将该节点添加至sync queue队列的尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); 
    return true;
}
```
在transferForSignal方法中，我们先使用CAS操作将当前节点的waitStatus状态由CONDTION设为0，如果修改不成功，
则说明该节点已经被CANCEL了，则我们直接返回，操作下一个节点；如果修改成功，
则说明我们已经将该节点从等待的条件队列中成功“唤醒”了，但此时该节点对应的线程并没有真正被唤醒，它还要和其他普通线程一样去争锁，
因此它将被添加到sync queue的末尾等待获取锁。  
<br/>
不过这里尤其注意的是，enq方法将node节点添加进队列时，返回的是node的前驱节点。  
<br/>
在将节点成功添加进sync queue中后，我们得到了该节点在sync queue中的前驱节点。我们前面说过，
在sync queque中的节点都要靠前驱节点去唤醒，所以，这里我们要做的就是将前驱节点的waitStatus设为Node.SIGNAL,
 这一点和shouldParkAfterFailedAcquire所做的工作类似：
``` java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
所不同的是，shouldParkAfterFailedAcquire将会向前查找，跳过那些被cancel的节点，
然后将找到的第一个没有被cancel的节点的waitStatus设成SIGNAL，最后再挂起。而在transferForSignal中，
当前Node所代表的线程本身就已经被挂起了，所以这里做的更像是一个复合操作——**只要前驱节点处于被取消的状态或者
无法将前驱节点的状态修成Node.SIGNAL，那我们就将Node所代表的线程唤醒**，但这个条件并不意味着当前lock处于可获取的状态，
有可能线程被唤醒了，但是锁还是被占有的状态，不过这样做至少是无害的，因为我们在线程被唤醒后还要去争锁，如果抢不到锁，
则大不了再次被挂起。  
<br/>
值得注意的是，transferForSignal是有返回值的，但是我们在这个方法中并没有用到，它将在signal()方法中被使用。  
<br/>
这里我们再总结一下signalAll()方法：  

- 将条件队列清空（只是令lastWaiter = firstWaiter = null，队列中的节点和连接关系仍然还存在）  
- 将条件队列中的头节点取出，使之成为孤立节点(nextWaiter,prev,next属性都为null)  
- 如果该节点处于被Cancelled了的状态，则直接跳过该节点（由于是孤立节点，则会被GC回收）  
- 如果该节点处于正常状态，则通过enq方法将它添加到sync queue的末尾  
- 判断是否需要将该节点唤醒(包括设置该节点的前驱节点的状态为SIGNAL)，如有必要，直接唤醒该节点
重复2-5，直到整个条件队列中的节点都被处理完  
#### 4.3 signal()
与signalAll()方法不同，signal()方法只会唤醒一个节点，对于AQS的实现来说，就是唤醒条件队列中第一个没有被Cancel的节点：  

``` java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```
首先依然是检查调用该方法的线程(即当前线程)是不是已经持有了锁，这一点和上面的signalAll()方法一样，所不一样的是，
接下来调用的是doSignal方法：  
##### 4.3.1 doSignal

``` java
private void doSignal(Node first) {
    do {
        // 将firstWaiter指向条件队列队头的下一个节点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 将条件队列原来的队头从条件队列中断开，则此时该节点成为一个孤立的节点
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```
这个方法也是一个do-while循环，目的是遍历整个条件队列，找到第一个没有被cancelled的节点，并将它添加到条件队列的末尾。
如果条件队列里面已经没有节点了，则将条件队列清空（firstWaiter=lasterWaiter=null）。  
<br/>
在这里，我们依然用的是transferForSignal方法，但是用到了它的返回值，只要节点被成功添加到sync queue中，
transferForSignal就返回true, 此时while循环的条件就不满足了，整个方法就结束了，即调用signal()方法，只会唤醒一个线程。  
<br/>
总结： 调用signal()方法会从当前条件队列中取出第一个没有被cancel的节点添加到sync队列的末尾。
#### 4.4 await()第二部分分析
前面我们已经分析了signal方法，它会将节点添加进sync queue队列中，并要么立即唤醒线程，要么等待前驱节点释放锁后将自己唤醒，
无论怎样，被唤醒的线程要从哪里恢复执行呢？当然是被挂起的地方呀，我们在哪里被挂起的呢？当然是调用了await方法的地方，
以await()方法为例：
``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 我们在这里被挂起了，被唤醒后，将从这里继续往下运行
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
这里值得注意的是，当我们被唤醒时，其实并不知道是因为什么原因被唤醒，有可能是因为其他线程调用了signal方法，
也有可能是因为当前线程被中断了。  
<br/>
但是，无论是被中断唤醒还是被signal唤醒，被唤醒的线程最后都将离开condition queue，进入到sync queue中。  
<br/>
随后，线程将在sync queue中利用进行acquireQueued方法进行“阻塞式”争锁，抢到锁就返回，抢不到锁就继续被挂起。因此，
当await()方法返回时，必然是保证了当前线程已经持有了lock锁。  
<br/>
另外有一点这里我们提前说明一下，这一点对于我们下面理解源码很重要，那就是：

>如果从线程被唤醒，到线程获取到锁这段过程中发生过中断，该怎么处理？  

我们前面分析中断的时候说过，中断对于当前线程只是个建议，由当前线程决定怎么对其做出处理。在acquireQueued方法中，
我们对中断是不响应的，只是简单的记录抢锁过程中的中断状态，并在抢到锁后将这个中断状态返回，交于上层调用的函数处理，
而这里“上层调用的函数”就是我们的await()方法。  
<br/>
那么await()方法是怎么对待这个中断的呢？这取决于：  

> 中断发生时，线程是否已经被signal过？  

如果中断发生时，当前线程并没有被signal过，则说明当前线程还处于条件队列中，属于正常在等待中的状态，
此时中断将导致当前线程的正常等待行为被打断，进入到sync queue中抢锁，因此，在我们从await方法返回后，
需要抛出InterruptedException，表示当前线程因为中断而被唤醒。  
<br/>
如果中断发生时，当前线程已经被signal过了，则说明这个中断来的太晚了，既然当前线程已经被signal过了，那么就说明在中断发生前，
它就已经正常地被从condition queue中唤醒了，所以随后即使发生了中断（注意，这个中断可以发生在抢锁之前，也可以发生在抢锁的过程中），
我们都将忽略它，仅仅是在await()方法返回后，再自我中断一下，补一下这个中断。就好像这个中断是在await()方法调用结束之后才发生的一样。
这里之所以要“补一下”这个中断，是因为我们在用Thread.interrupted()方法检测是否发生中断的同时，会将中断状态清除，
因此如果选择了忽略中断，则应该在await()方法退出后将它设成原来的样子。  
<br/>
关于“这个中断来的太晚了”这一点如果大家不太容易理解的话，这里打个比方：这就好比我们去饭店吃饭，都快吃完了，
有一个菜到现在还没有上，于是我们常常会把服务员叫来问：这个菜有没有在做？要是还没做我们就不要了。然后服务员会跑到厨房去问，
之后跑回来说：对不起，这个菜已经下锅在炒了，请再耐心等待一下。这里，这个“这个菜我们不要了”(发起的中断)就来的太晚了，
因为菜已经下锅了(已经被signal过了)。  
<br/>
理清了上面的概念，我们再来看看await()方法是怎么做的，它用中断模式interruptMode这个变量记录中断事件，该变量有三个值：

- 0 ： 代表整个过程中一直没有中断发生。  
- THROW_IE ： 表示退出await()方法时需要抛出InterruptedException，这种模式对应于中断发生在signal之前  
- REINTERRUPT ： 表示退出await()方法时只需要再自我中断以下，这种模式对应于中断发生在signal之后，即中断来的太晚了。  

``` java
private static final int REINTERRUPT =  1;
private static final int THROW_IE    = -1;
```
接下来我们就从线程被唤醒的地方继续往下走，一步步分析源码，贴上await源码：  
``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 我们在这里被挂起了，被唤醒后，将从这里继续往下运行
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

```

##### 4.4.1 中断发生时，线程还没有被signal过
线程被唤醒后，我们将首先使用checkInterruptWhileWaiting方法检测中断的模式:  
``` java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
}
```
这里假设已经发生过中断，则Thread.interrupted()方法必然返回true，接下来就是用transferAfterCancelledWait进一步判断是否发生了signal：  

``` java
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```
上面已经说过，判断一个node是否被signal过，一个简单有效的方法就是判断它是否离开了condition queue, 进入到sync queue中。  
<br/>
换句话说，只要一个节点的waitStatus还是Node.CONDITION，那就说明它还没有被signal过。  

由于现在我们分析情况1，则当前节点的waitStatus必然是Node.CONDITION，则会成功执行compareAndSetWaitStatus(node, Node.CONDITION, 0)，
将该节点的状态设置成0，然后调用enq(node)方法将当前节点添加进sync queue中，然后返回true。  
<br/>
这里值得注意的是，我们此时并没有断开node的nextWaiter，所以最后一定不要忘记将这个链接断开。  
<br/>
再回到transferAfterCancelledWait调用处，可知，由于transferAfterCancelledWait将返回true，
现在checkInterruptWhileWaiting将返回THROW_IE，这表示我们在离开await方法时应当要抛出THROW_IE异常。  
<br/>
再回到checkInterruptWhileWaiting的调用处，interruptMode现在为THROW_IE，则我们将执行break，跳出while循环。  
<br/>
接下来我们将执行acquireQueued(node, savedState)进行争锁，注意，这里传入的需要获取锁的重入数量是savedState，
即之前释放了多少，这里就需要再次获取多少：   

``` java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // 如果线程获取不到锁，则将在这里被阻塞住
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```
acquireQueued是一个阻塞式的方法，获取到锁则退出，获取不到锁则会被挂起。该方法只有在最终获取到了锁后，才会退出，
并且退出时会返回当前线程的中断状态，如果我们在获取锁的过程中又被中断了，则会返回true，否则会返回false。
但是其实这里返回true还是false已经不重要了，因为前面已经发生过中断了，我们就是因为中断而被唤醒的不是吗？所以无论如何，
我们在退出await()方法时，必然会抛出InterruptedException。  
<br/>
我们这里假设它获取到了锁了，则它将回到上面的调用处，由于我们这时的interruptMode = THROW_IE，则会跳过if语句。接下来我们将执行：  

``` java
if (node.nextWaiter != null) 
    unlinkCancelledWaiters();
```
上面我们说过，当前节点的nextWaiter是有值的，它并没有和原来的condition队列断开，这里我们已经获取到锁了，前面在独占锁的获取中的分析，
我们通过setHead方法已经将它的thread属性置为null，从而将当前线程从sync queue"移除"了，接下来应当将它从condition队列里面移除。
由于condition队列是一个单向队列，我们无法获取到它的前驱节点，所以只能从头开始遍历整个条件队列，然后找到这个节点，再移除它。  
<br/>
然而，事实上呢，我们并没有这么做。因为既然已经必须从头开始遍历链表了，我们就干脆一次性把链表中所有没有在等待的节点都拿出去，
所以这里调用了unlinkCancelledWaiters方法，该方法我们在前面await()第一部分的分析的时候已经讲过了，它就是简单的遍历链表，
找到所有waitStatus不为CONDITION的节点，并把它们从队列中移除。  
<br/>
节点被移除后，接下来就是最后一步了——汇报中断状态：

``` java
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```
这里我们的interruptMode=THROW_IE，说明发生了中断，则将调用reportInterruptAfterWait：


``` java
private void reportInterruptAfterWait(int interruptMode) throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}

```
可以看出，在interruptMode=THROW_IE时，我们就是简单的抛出了一个InterruptedException。  
<br/>
至此，情况1（中断发生于signal之前）我们就分析完了，这里我们简单总结一下：  

- 1. 线程因为中断，从挂起的地方被唤醒  
- 2. 随后，我们通过transferAfterCancelledWait确认了线程的waitStatus值为Node.CONDITION，说明并没有signal发生过  
- 3. 然后我们修改线程的waitStatus为0，并通过enq(node)方法将其添加到sync queue中  
- 4. 接下来线程将在sync queue中以阻塞的方式获取，如果获取不到锁，将会被再次挂起  
- 5. 线程在sync queue中获取到锁后，将调用unlinkCancelledWaiters方法将自己从条件队列中移除，该方法还会顺便移除其他取消等待的锁
- 6. 最后我们通过reportInterruptAfterWait抛出了InterruptedException  

<br/>

由此可以看出，一个调用了await方法挂起的线程在被中断后不会立即抛出InterruptedException，而是会被添加到sync queue中去争锁，如果争不到，还是会被挂起；
只有争到了锁之后，该线程才得以从sync queue和condition queue中移除，最后抛出InterruptedException。  
<br/>
所以说，一个调用了await方法的线程，即使被中断了，它依旧还是会被阻塞住，直到它获取到锁之后才能返回，
并在返回时抛出InterruptedException。中断对它意义更多的是体现在将它从condition queue中移除，加入到sync queue中去争锁，
从这个层面上看，中断和signal的效果其实很像，所不同的是，在await()方法返回后，如果是因为中断被唤醒，
则await()方法需要抛出InterruptedException异常，表示是它是被非正常唤醒的（正常唤醒是指被signal唤醒）。  
##### 4.4.2 中断发生时，线程已经被signal过了
这种情况对应于“中断来的太晚了”，即REINTERRUPT模式，我们在拿到锁退出await()方法后，只需要再自我中断一下，
不需要抛出InterruptedException。  
<br/>
值得注意的是这种情况其实包含了两个子情况：  

- 情况1：被唤醒时，已经发生了中断，但此时线程已经被signal过了  
- 情况2：被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断  

<br/>

下面我们分别来分析：  

**情况1：被唤醒时，已经发生了中断，但此时线程已经被signal过了**  
对于这种情况，与前面中断发生于signal之前的主要差别在于transferAfterCancelledWait方法：  

``` java
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) { //线程A执行到这里，CAS操作将会失败
        enq(node);
        return true;
    }
    // 由于中断发生前，线程已经被signal了，则这里只需要等待线程成功进入sync queue即可
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```
在这里，由于signal已经发生过了，则由我们之前分析的signal方法可知，此时当前节点的waitStatus必定不为Node.CONDITION，
他将跳过if语句。此时当前线程可能已经在sync queue中，或者正在进入到sync queue的路上。  
<br/>
为什么这里会出现“正在进入到sync queue的路上”的情况呢？ 这里我们解释下：  
假设当前线程为线程A, 它被唤醒之后检测到发生了中断，来到了transferAfterCancelledWait这里，
而另一个线程B在这之前已经调用了signal方法，该方法会调用transferForSignal将当前线程添加到sync queue的末尾：  

``` java
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)) // 线程B执行到这里，CAS操作将会成功
        return false; 
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
因为线程A和线程B是并发执行的，而这里我们分析的是“中断发生在signal之后”，则此时，
线程B的compareAndSetWaitStatus先于线程A执行。这时可能出现线程B已经成功修改了node的waitStatus状态，
但是还没来得及调用enq(node)方法，线程A就执行到了transferAfterCancelledWait方法，此时它发现waitStatus已经不是Condition，
但是其实当前节点还没有被添加到sync node队列中，因此，它接下来将通过自旋，等待线程B执行完transferForSignal方法。  
<br/>
线程A在自旋过程中会不断地判断节点有没有被成功添加进sync queue，判断的方法就是isOnSyncQueue：  

``` java
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    return findNodeFromTail(node);
}
```
该方法很好理解，只要waitStatus的值还为Node.CONDITION，则它一定还在condtion队列中，自然不可能在sync里面；
而每一个调用了enq方法入队的线程：

``` java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) { //即使这一步失败了next.prev一定是有值的
                t.next = node; // 如果t.next有值，说明上面的compareAndSetTail方法一定成功了，则当前节点成为了新的尾节点
                return t; // 返回了当前节点的前驱节点
            }
        }
    }
}
```
哪怕在设置compareAndSetTail这一步失败了，它的prev必然也是有值的，因此这两个条件只要有一个满足，
就说明节点必然不在sync queue队列中。  
<br/>
另一方面，如果node.next有值，则说明它不仅在sync queue中，并且在它后面还有别的节点，则只要它有值，该节点必然在sync queue中。  
如果以上都不满足，说明这里出现了尾部分叉的情况，我们就从尾节点向前寻找这个节点：  
``` java
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```
这里当然还是有可能出现从尾部反向遍历找不到的情况，但是不用担心，我们(isOnSyncQueue)还在while循环(transferAfterCancelledWait的while)中，
无论如何，节点最后总会入队成功的。最终，transferAfterCancelledWait将返回false。  
<br/>
再回到transferAfterCancelledWait调用处：  

``` java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```
则这里，由于transferAfterCancelledWait返回了false，则checkInterruptWhileWaiting方法将返回REINTERRUPT，
这说明我们在退出该方法时只需要再次中断。  
<br/>
再回到checkInterruptWhileWaiting方法的调用处：  

``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) //我们在这里！！！
            break;
    }
    //当前interruptMode=REINTERRUPT，无论这里是否进入if体，该值不变
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) 
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
此时，interruptMode的值为REINTERRUPT，我们将直接跳出while循环。  
<br/>
接下来就和上面的情况1一样了，我们依然还是去争锁，这一步依然是阻塞式的，获取到锁则退出，获取不到锁则会被挂起。  
<br/>
另外由于现在interruptMode的值已经为REINTERRUPT，因此无论在争锁的过程中是否发生过中断interruptMode的值都还是REINTERRUPT。  
<br/>
接着就是将节点从condition queue中剔除，与情况1不同的是，在signal方法成功将node加入到sync queue时，
该节点的nextWaiter已经是null了，所以这里这一步不需要执行。  
<br/>
再接下来就是报告中断状态了：  

``` java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```
注意，这里并没有抛出中断异常，而只是将当前线程再中断一次。  
<br/>
至此，情况1（被唤醒时，已经发生了中断，但此时线程已经被signal过了）我们就分析完了，这里我们简单总结一下：  

- 线程从挂起的地方被唤醒，此时既发生过中断，又发生过signal  
- 随后，我们通过transferAfterCancelledWait确认了线程的waitStatus值已经不为Node.CONDITION，说明signal发生于中断之前  
- 然后，我们通过自旋的方式，等待signal方法执行完成，确保当前节点已经被成功添加到sync queue中  
- 接下来线程将在sync queue中以阻塞的方式获取锁，如果获取不到，将会被再次挂起  
- 最后我们通过reportInterruptAfterWait将当前线程再次中断，但是不会抛出InterruptedException  
<br/>
<br/>
**情况2：被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断**  

这种情况就比上面的情况简单一点了，既然被唤醒时没有发生中断，那基本可以确信线程是被signal唤醒的，
但是不要忘记还存在“假唤醒”这种情况，因此我们依然还是要检测被唤醒的原因。  
<br/>
那么怎么区分到底是假唤醒还是因为是被signal唤醒了呢？  
<br/>
如果线程是因为signal而被唤醒，则由前面分析的signal方法可知，线程最终都会离开condition queue 进入sync queue中，
所以我们只需要判断被唤醒时，线程是否已经在sync queue中即可:  

``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);  // 我们在这里，线程将在这里被唤醒
        // 由于现在没有发生中断，所以interruptMode目前为0
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
线程被唤醒时，暂时还没有发生中断，所以这里interruptMode = 0, 表示没有中断发生，所以我们将继续while循环，
这时我们将通过isOnSyncQueue方法判断当前线程是否已经在sync queue中了。由于已经发生过signal了，
则此时node必然已经在sync queue中了，所以isOnSyncQueue将返回true,我们将退出while循环。  
<br/>
不过这里插一句，如果isOnSyncQueue检测到当前节点不在sync queue中，则说明既没有发生中断，也没有发生过signal，
则当前线程是被“假唤醒”的，那么我们将再次进入循环体，将线程挂起。  
<br/>
退出while循环后接下来还是利用acquireQueued争锁，因为前面没有发生中断，则interruptMode=0，这时，
如果在争锁的过程中发生了中断，则acquireQueued将返回true，则此时interruptMode将变为REINTERRUPT。  
<br/>
接下是判断node.nextWaiter != null，由于在调用signal方法时已经将节点移出了队列，所有这个条件也不成立。  
<br/>
最后就是汇报中断状态了，此时interruptMode的值为REINTERRUPT，说明线程在被signal后又发生了中断，这个中断发生在抢锁的过程中，
这个中断来的太晚了，因此我们只是再次自我中断一下。  
<br/>
至此，情况2（被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断）我们就分析完了，这种情况和情况1很像，
区别就是一个是在唤醒后就被发现已经发生了中断，一个在唤醒后没有发生中断，但是在抢锁的过成中发生了中断，但无论如何，
这两种情况都会被归结为“中断来的太晚了”，中断模式为REINTERRUPT，情况2的总结如下：  

- 线程被signal方法唤醒，此时并没有发生过中断  
- 因为没有发生过中断，我们将从checkInterruptWhileWaiting处返回，此时interruptMode=0  
- 接下来我们回到while循环中，因为signal方法保证了将节点添加到sync queue中，此时while循环条件不成立，循环退出  
- 接下来线程将在sync queue中以阻塞的方式获取，如果获取不到锁，将会被再次挂起  
- 线程获取到锁返回后，我们检测到在获取锁的过程中发生过中断，并且此时interruptMode=0，这时，我们将interruptMode修改为REINTERRUPT  
- 最后我们通过reportInterruptAfterWait将当前线程再次中断，但是不会抛出InterruptedException  
<br/>
这里我们再总结以下情况2（中断发生时，线程已经被signal过了），这种情况对应于中断发生signal之后，
我们不管这个中断是在抢锁之前就已经发生了还是抢锁的过程中发生了，只要它是在signal之后发生的，我们就认为它来的太晚了，
我们将忽略这个中断。因此，从await()方法返回的时候，我们只会将当前线程重新中断一下，而不会抛出中断异常。  
##### 4.4.3 一直没有中断发生
这种情况就更简单了，它的大体流程和上面的情况2差不多，只是在抢锁的过程中也没有发生异常，则interruptMode为0，没有发生过中断，
因此不需要汇报中断。则线程就从await()方法处正常返回。
#### 4.5 await()总结
至此，我们总算把await()方法完整的分析完了，这里我们对整个方法做出总结：  

- 进入await()时必须是已经持有了锁  
- 离开await()时同样必须是已经持有了锁  
- 调用await()会使得当前线程被封装成Node扔进条件队列，然后释放所持有的锁  
- 释放锁后，当前线程将在condition queue中被挂起，等待signal或者中断  
- 线程被唤醒后会将会离开condition queue进入sync queue中进行抢锁  
- 若在线程抢到锁之前发生过中断，则根据中断发生在signal之前还是之后记录中断模式  
- 线程在抢到锁后进行善后工作（离开condition queue, 处理中断异常）  
- 线程已经持有了锁，从await()方法返回  

![](/images/posts/多线程/源码系列/AQS源码-Condition-await()总结.png)  
在这一过程中我们尤其要关注中断，如前面所说，中断和signal所起到的作用都是将线程从condition queue中移除，
加入到sync queue中去争锁，所不同的是，signal方法被认为是正常唤醒线程，中断方法被认为是非正常唤醒线程，
如果中断发生在signal之前，则我们在最终返回时，应当抛出InterruptedException；
如果中断发生在signal之后，我们就认为线程本身已经被正常唤醒了，这个中断来的太晚了，我们直接忽略它，并在await()返回时再自我中断一下，
这种做法相当于将中断推迟至await()返回时再发生。  

## 5. awaitUninterruptibly()
在前面我们分析的await()方法中，中断起到了和signal同样的效果，但是中断属于将一个等待中的线程非正常唤醒，可能即使线程被唤醒后，
也抢到了锁，但是却发现当前的等待条件并没有满足，则还是得把线程挂起。因此我们有时候并不希望await方法被中断，
awaitUninterruptibly()方法即实现了这个功能：  

``` java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true; // 发生了中断后线程依旧留在了condition queue中，将会再次被挂起
    }
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```
首先，从方法签名上就可以看出，这个方法不会抛出中断异常，我们拿它和await()方法对比一下：  


``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())  // 不同之处
        throw new InterruptedException(); // 不同之处
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;  // 不同之处
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)  // 不同之处
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)  // 不同之处
        interruptMode = REINTERRUPT;  // 不同之处
    if (node.nextWaiter != null)  // 不同之处
        unlinkCancelledWaiters(); // 不同之处
    if (interruptMode != 0) // 不同之处
        reportInterruptAfterWait(interruptMode); // 不同之处
}
```
由此可见，awaitUninterruptibly()全程忽略中断，即使是当前线程因为中断被唤醒，该方法也只是简单的记录中断状态，
然后再次被挂起（因为并没有并没有任何操作将它添加到sync queue中）  
<br/>
要使当前线程离开condition queue去争锁，则必须是发生了signal事件。  
<br/>
最后，当线程在获取锁的过程中发生了中断，该方法也是不响应，只是在最终获取到锁返回时，再自我中断一下。可以看出，
该方法和“中断发生于signal之后的”REINTERRUPT模式的await()方法很像。  
<br/>
至此，该方法我们就分析完了，如果你之前await()方法已经弄懂了，这个awaitUninterruptibly()方法就很容易理解了。它的核心思想是：  
- 中断虽然会唤醒线程，但是不会导致线程离开condition queue，如果线程只是因为中断而被唤醒，则他将再次被挂起  
- 只有signal方法会使得线程离开condition queue  
- 调用该方法时或者调用过程中如果发生了中断，仅仅会在该方法结束时再自我中断以下，不会抛出InterruptedException  
## 6. awaitNanos(long nanosTimeout)
前面我们看的方法，无论是await()还是awaitUninterruptibly()，它们在抢锁的过程中都是阻塞式的，即一直到抢到了锁才能返回，
否则线程还是会被挂起，这样带来一个问题就是线程如果长时间抢不到锁，就会一直被阻塞，因此我们有时候更需要带超时机制的抢锁，
这一点和带超时机制的wait(long timeout)是很像的，我们直接来看源码：  

``` java
public final long awaitNanos(long nanosTimeout) throws InterruptedException {
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);*/
    final long deadline = System.nanoTime() + nanosTimeout;
    /*int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);
            break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
        nanosTimeout = deadline - System.nanoTime();
    }
    /*if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    return deadline - System.nanoTime();
}
```
该方法几乎和await()方法一样，只是多了超时时间的处理，我们上面已经把和await()方法相同的部分注释起来了，
只留下了不同的部分，这样它们的区别就变得更明显了。  
<br/>
该方法的主要设计思想是，如果设定的超时时间还没到，我们就将线程挂起；超过等待的时间了，
我们就将线程从condtion queue转移到sync queue中。
注意这里对于超时时间有一个小小的优化——当设定的超时时间很短时（小于spinForTimeoutThreshold的值），我们就是简单的自旋，
而不是将线程挂起，以减少挂起线程和唤醒线程所带来的时间消耗。  
<br/>

不过这里还有一处值得注意，就是awaitNanos(0)的意义，wait(0)的含义是无限期等待，
而我们在awaitNanos(long nanosTimeout)方法中是怎么处理awaitNanos(0)的呢？  

``` java
if (nanosTimeout <= 0L) {
    transferAfterCancelledWait(node);
    break;
}
```
从这里可以看出，如果设置的等待时间本身就小于等于0，当前线程是会直接从condition queue中转移到sync queue中的，并不会被挂起，
也不需要等待signal，这一点确实是更复合逻辑。如果需要线程只有在signal发生的条件下才会被唤醒，则应该用上面的awaitUninterruptibly()方法。  
## 7. await(long time, TimeUnit unit)
看完awaitNanos(long nanosTimeout)再看await(long time, TimeUnit unit)方法就更简单了，
它就是在awaitNanos(long nanosTimeout)的基础上多了对于超时时间的时间单位的设置，但是在内部实现上还是会把时间转成纳秒去执行，
这里我们直接拿它和上面的awaitNanos(long nanosTimeout)方法进行对比，只给出不同的部分：  

``` java
public final boolean await(long time, TimeUnit unit) throws InterruptedException {
    long nanosTimeout = unit.toNanos(time);
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;*/
    /*boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {*/
            timedout = transferAfterCancelledWait(node);
            /*break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    return !timedout;
}
```
可以看出，这两个方法主要的差别就体现在返回值上面，awaitNanos(long nanosTimeout)的返回值是剩余的超时时间，如果该值大于0，
说明超时时间还没到，则说明该返回是由signal行为导致的，
而await(long time, TimeUnit unit)的返回值就是transferAfterCancelledWait(node)的值，我们知道，如果调用该方法时，
node还没有被signal过则返回true，node已经被signal过了，则返回false。因此当await(long time, TimeUnit unit)方法返回true，
则说明在超时时间到之前就已经发生过signal了，该方法的返回是由signal方法导致的而不是超时时间。  
<br/>
综上，调用await(long time, TimeUnit unit)其实就等价于调用awaitNanos(unit.toNanos(time)) > 0 方法。  
## 8. awaitUntil(Date deadline)
awaitUntil(Date deadline)方法与上面的几种带超时的方法也基本类似，所不同的是它的超时时间是一个绝对的时间，
我们直接拿它来和上面的await(long time, TimeUnit unit)方法对比：  

``` java
public final boolean awaitUntil(Date deadline) throws InterruptedException {
    long abstime = deadline.getTime();
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        if (System.currentTimeMillis() > abstime) {
            /*timedout = transferAfterCancelledWait(node);
            break;
        }*/
        LockSupport.parkUntil(this, abstime);
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
    }
    /*
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;*/
}
```
可见，这里大段的代码都是重复的，区别就是在超时时间的判断上使用了绝对时间，
其实这里的deadline就和awaitNanos(long nanosTimeout)以及await(long time, TimeUnit unit)内部的deadline变量是等价的，
另外就是在这个方法中，没有使用spinForTimeoutThreshold进行自旋优化，因为一般调用这个方法，目的就是设定一个较长的等待时间，
否则使用上面的相对时间会更方便一点。


