---
layout: post
title: 阻塞队列之PriorityBlockingQueue源码
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---

>PriorityBlockingQueue 是一个支持优先级的无界阻塞队列，默认情况下元素采取自然顺序升序排列，
也可以自定义类实现compareTo()方法来指定元素排序规则，需要注意的是不能保证同优先级元素的顺序。  

>所谓无无界队列就是没有容量限制，可以"无限"的扩张，当然也不是无休止的扩张，这个还是有最大的限制的。

## 1. 继承体系
![](/images/posts/多线程/源码系列/阻塞队列-PriorityBlockingQueue之类图.png)  
PriorityBlockingQueue 实现了BlockingQueue接口，该接口中定义了阻塞的方法接口。  
PriorityBlockingQueue 继承了AbstractQueue，具有了队列的行为。  
PriorityBlockingQueue 实现了Serializable接口,可以序列化。  
## 2. 数据结构
``` java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = 5595510919245408276L;
    
     //队列默认大小
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

	 //队列最大容量，超过该容量抛oom异常
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //队列，数组实现 不序列化
    private transient Object[] queue;

     //队列元素个数 不序列化
    private transient int size;

  
    private transient Comparator<? super E> comparator;

     //用于队列操作的锁
    private final ReentrantLock lock;

     //队列非空条件
    private final Condition notEmpty;

    //队列扩容的 "锁" 不序列化
    private transient volatile int allocationSpinLock;

	//用于序列化
    private PriorityQueue<E> q;
    ...
 }
```
ArrayBlockingQueue，LinkedBlockingQueue 中通过指定大小来确定队列的大小，队列大小一旦确定后就不会改变，
同时队列是否入队或者出队由两个条件来控制（notEmpty 和notFull ）,因此它们都是有界的阻塞队列。  
<br/>
在PriorityBlockingQueue 我们看到只有notEmpty 条件，没有notFull 条件，同时也有默认的队列大小，
也就是说PriorityBlockingQueue 没有队满的概念，当元素个数超过队列长度以后，那么就进行扩容，当达到最大的队列容量后就不能继续入队了，
否则就会抛异常，而不是前面两个阻塞队列那样，等到notFull 条件满足，具体我们可以在PriorityBlockingQueue。  
<br/>
PriorityBlockingQueue 中通过一个可重入锁来控制入队和出队行为，这个和ArrayBlockingQueue 中是一致的，
LinkedBlockingQueue 中对入队和出队用了两个不同的可重入锁来控制。  
<br/>
PriorityBlockingQueue 底层还是通过数组来实现的。
## 3. 二叉堆
PriorityBlockingQueue 是一种优先级的队列，其入队和出队会按照优先级进行排序，是基于优先级的FIFO.  
<br/>
PriorityBlockingQueue 是基于数组来存储的，但是其数据结构是二叉堆(分为最大堆和最小堆)，这里二叉堆被用来做优先级队列。  

二叉堆本质是一颗二叉树：  

- 二叉堆是一种特殊的堆，二叉堆是完全二叉树或者是近似完全二叉树  
- 如果节点在数组中的位置是i(i是节点在数组中的下标), 则i节点对应的子节点在数组中的位置分别是 2i + 1 和 2i +2，
同时i的父节点的位置为 (i-1)/2（i从0 开始）  
- 通过一次上浮可以把符合条件的元素放到堆顶，反复进行上浮操作，可以将整个堆进行有序化  
- 下沉操作可以把替代堆顶后的元素放到该放的位置，同时堆顶元素是符合条件的元素（前提是二叉堆已经是有序的）  
## 4. 构造方法
#### 4.1 默认构造
``` java
public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
```
默认构造方法，会调用其它构造方法，指定一个默认的容量，同时比较器Comparator 为null  
#### 4.2 指定容量构造方法
``` java
    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

```
#### 4.3 指定容量，指定比较器
``` java
    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.comparator = comparator;
        //生成队列
        this.notEmpty = lock.newCondition();
        this.queue = new Object[initialCapacity];
    }

```
## 5. 入队
#### 5.1 add(E e)
将指定的元素插入到此队列中，在成功时返回 true
``` java
    public boolean add(E e) {
        return offer(e);
    }

```
#### 5.2 offer(E e)
将指定的元素插入到此队列中，在成功时返回 true，在前面的add 中，内部调用了offer 方法，我们也可以直接调用offer方法来完成入队操作。  
``` java
    public boolean offer(E e) {
        //队列中的元素不能为空
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        int n, cap;
        Object[] array;
        //元素个数超出队列长度，则进行扩容
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);
        try {
            //比较器
            Comparator<? super E> cmp = comparator;
            //如果没有设置比较器，则进行自然顺序的插入
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                //使用指定比较器的插入
                siftUpUsingComparator(n, e, array, cmp);
            size = n + 1;
            //入队后  notEmpty 条件满足，唤醒阻塞在notEmpty 条件上的一个线程
            notEmpty.signal();
        } finally {
            //释放锁
            lock.unlock();
        }
        return true;
    }

```
总结：  
1. 判断元素是否未空  
2. 判断容器是否需要扩容，如果是则扩容  
3. 入队操作  
4. Condition 释放信号  
#### 5.3 tryGrow
``` java
private void tryGrow(Object[] array, int oldCap) {
    //释放锁
    lock.unlock(); 
    Object[] newArray = null;
    // // 当前没有线程在执行扩容 && 原子更新扩容标识为 1
    if (allocationSpinLock == 0 && UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset, 0, 1)) {
        try {
            //1）旧容量小于 64，执行【双倍+2】扩容；
            //2）旧容量大于等于 64，执行1.5倍向下取整扩容
            int newCap = oldCap + ((oldCap < 64) ? (oldCap + 2) : (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {//新容量超出最大量
                //旧容量增加1
                int minCap = oldCap + 1;
                //如果此时旧容量内存溢出 或 大于最大容量 抛出异常
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;//写入最大容量
            }
            //queue == array 这里保证 queue还未被修改
            if (newCap > oldCap && queue == array)__
                newArray = new Object[newCap];
        } finally {
            //还原
            allocationSpinLock = 0;
        }
    }
    //其它线程对队列进行了改动，放弃扩容，等待其他线程扩容完成
    if (newArray == null)
        Thread.yield(); 
    //重新加锁，准备回到offer 中    
    lock.lock();
    if (newArray != null && queue == array) {
         //扩容，复制内容到新数组
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```
总结：  
1. 释放了可重入锁，此时其它线程可以操控队列  
2. 如果allocationSpinLock=0 ，则cas 设置成为1  
3. 如果超出最大容量，则抛oom(outOfMemory)  
4. 如果队列没有被修改，则扩容  
5. 准备回到offer 方法中，重新加锁，如果获取到锁，其它线程无法修改队列  
6. 如果期间队列没有被修改，那么扩容，复制队列元素到新队列  
7. 还原allocationSpinLock  
<br/>
这里的allocationSpinLock 其实相当于锁的功能，因为在该方法中，释放掉了锁，那么其它线程可能就会操作队列，那么也可能进行扩容操作，
为了保证扩容的线程安全，那么就用allocationSpinLock 来进行记录，来保证只有一个线程能执行扩容代码。  
<br/>
通过判断 queue == array 是否相等（数组中的每个元素都是否相等），来判断是否其它线程对队列元素进行了修改，
如果其它元素对队列进行了修改，那么就会放弃扩容，因此才会看到在 offer 中通过while 循环来判断是否真正需要扩容  
<br/>
从offer 中进入到tryGrow 中释放了锁，因此最后需要重新获取锁，获取锁后，其它线程将不能操作队列，此时再次判断是否能扩容，
如果是则进行扩容，复制队列元素到新队列中，完毕。  
#### 5.4 siftUpComparable
``` java
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    //元素自身默认比较器
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        //元素x的父节点的位置 (k-1)/2
        int parent = (k - 1) >>> 1;
        //父节点
        Object e = array[parent];
        //判断是否将x元素存在位置k,当前元素和父节点比较，如果当前节点大于父节点，退出循环
        if (key.compareTo((T) e) >= 0)
            break;
        //当前节点比父节点小，父节点元素下沉，    
        array[k] = e;
        //从父节点位置开始，判断是否将x元素存在位置k
        k = parent;
    }
    //找到位置k,将元素x 存放在该位置
    array[k] = key;
}

```
对于siftUpUsingComparator其实是一样的，只是比较器变了而已。
#### 5.5 offer(E, long, TimeUnit)
``` java
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e); // never need to block
}
```
这个方法在ArrayBlockingQueue，LinkedBlockingQueue 中都进行了超时等待，而这里实际上并没有进行超时等待。
因为PriorityBlockingQueue 是无界队列的概念，不会“队满”，实际当到达队列最大值后，就抛oom异常了，因此这点在使用优先队列的时候，
需要注意。  
#### 5.6 put(E e)
将指定的元素插入此队列中,队列达到最大值，则抛oom异常  
``` java
public void put(E e) {
    offer(e); // never need to block
}

```
## 6. 出队
#### 6.1 poll
获取并移除此队列的头，如果此队列为空，则返回 null  
``` java
public E poll() {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //返回出队元素
        return dequeue();
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
#### 6.2 dequeue
``` java
private E dequeue() {
    int n = size - 1;
    //队列为空，返回null
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        //堆顶就是我们需要的元素
        E result = (E) array[0];
        //堆中最后一个元素
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        //下沉操作
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}

```
#### 6.3 siftDownComparable

``` java
    private static <T> void siftDownComparable(int k, T x, Object[] array, int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            int half = n >>> 1;
            while (k < half) {
                // 2*k+1 表示的k的左孩子的位置
                int child = (k << 1) + 1; // assume left child is least
                Object c = array[child];
                // 2*k+2 右孩子位置
                int right = child + 1;
				//取左右孩子中元素值较小的值（这里的较小，是通过比较器来定义的较小）
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    c = array[child = right];
                //x 比左右孩子都小，那么不用继续下沉了    
                if (key.compareTo((T) c) <= 0)
                    break;
                //下沉    
                array[k] = c;
                k = child;
            }
            array[k] = key;
        }
    }

```
n是我们的队列中元素个数-1,因为数组是总下标0开始存的，因此n-1 就是最后一个元素的下标。对于任何一个角标i来说，
2i+1 就是左孩子的下标，因为二叉树是完全二叉树结构，因此有右孩子就必定有左孩子，有左孩子不一定有右孩子，
没左孩子必定没右孩子，因此2i+1 <=n,即 i<=(n-1)/2,当然 i<=n/2,因为叶子节点没有孩子，不需要再下沉了，
进行i<n/2就可以了（这里的i 就相当于k）
#### 6.4 poll(long timeout, TimeUnit unit)
获取并移除此队列的头部，在指定的等待时间前等待。  

``` java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //可中断获取锁
        lock.lockInterruptibly();
        E result;
        try {
            超时等待
            while ( (result = dequeue()) == null && nanos > 0)
                nanos = notEmpty.awaitNanos(nanos);
        } finally {
            lock.unlock();
        }
        return result;
    }

```
#### 6.5 take()
获取并移除此队列的头部，在元素变得可用之前一直等待  

``` java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            //如果队列为空，则阻塞在notEmpty条件上
            while ( (result = dequeue()) == null)
                notEmpty.await();
        } finally {
            lock.unlock();
        }
        return result;
    }

```
#### 6.6 peek
调用此方法，可以返回队头元素，但是元素并不出队。

``` java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (size == 0) ? null : (E) queue[0];
        } finally {
            lock.unlock();
        }
    }

```
## 7 回顾：用集合初始化PriorityBlockingQueue

``` java
    public PriorityBlockingQueue(Collection<? extends E> c) {
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        //是否需要将堆进行有序化
        boolean heapify = true;
        //扫描null 值，保证队列中不会有null 元素
        boolean screen = true;
        if (c instanceof SortedSet<?>) {
            
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            //SortedSet 本身是有序的，因此不用进行堆有序化
            heapify = false;
        }
        else if (c instanceof PriorityBlockingQueue<?>) {
            PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            //PriorityBlockingQueue 本身就不会存null 值，因此不用再次扫描
            screen = false;
            //如果已经是本身类结构，那么也无需再次堆有序化
            if (pq.getClass() == PriorityBlockingQueue.class)
                heapify = false;
        }
        Object[] a = c.toArray();
        int n = a.length;
        //拷贝元素
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, n, Object[].class);
        //扫描集合，不允许出现null 
        if (screen && (n == 1 || this.comparator != null)) {
            for (int i = 0; i < n; ++i)
                if (a[i] == null)
                    throw new NullPointerException();
        }
        this.queue = a;
        this.size = n;
        if (heapify)
            heapify(); //堆有序化
    }

```
从集合中初始化PriorityBlockingQueue，需要进行判断  
1、是否需要进行有序化，PriorityBlockingQueue，SortedSet 本身有序，无需再进行有序化  
2、是否进行集合扫描，保证队列中不存储null 值元素  
#### 7.1 heapify

``` java
private void heapify() {
    Object[] array = queue;
    int n = size;
    //非叶子节点并且编号最大的节点
    int half = (n >>> 1) - 1;
    Comparator<? super E> cmp = comparator;
    if (cmp == null) {
        //对每个元素进行下沉操作
        for (int i = half; i >= 0; i--)
            siftDownComparable(i, (E) array[i], array, n);
    }
    else {
        for (int i = half; i >= 0; i--)
            siftDownUsingComparator(i, (E) array[i], array, n, cmp);
    }
}
```
在前面二叉堆我们说了，有序化，就是不断的进行上浮操作，而上浮从非叶子节点并且编号最大的节点开始调整，而这个编号怎么求呢，
就是：n/2-1。  
<br/>
这里我们看到，实际上进行的是下沉操作，并不是上浮，其实这样是一样的，其思想就是：从从非叶子节点并且编号最大的节点开始向上调整，
依次对元素进行下沉，那么该元素下沉，那么其它元素自然也就上浮了，只是看待的元素不一样而已。
## 8. 序列化
PriorityBlockingQueue 中的属性大部分都被transient 修饰，导致了不能进行默认的序列化，那么必然就需要进行手动序列化，
这个我们可以从writeObject 中得出答案  
``` java
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
    lock.lock();
    try {
        //生成一个新的队列
        q = new PriorityQueue<E>(Math.max(size, 1), comparator);
        //将队列中的元素添加到新的队列中
        q.addAll(this);
        //序列化新的队列
        s.defaultWriteObject();
    } finally {
        q = null;
        lock.unlock();
    }
}
```
这么做的用意就在于，队列的容量 >= 队列中元素的个数，为了不把没必要的null 值序列化，因此就重新生成一个队列，
避免过多的null 值被序列化，这种思想其实在前面各种集合框架中都用到了。
## 9. 总结
- PriorityBlockingQueue 是基于二叉堆来实现的，二叉堆底层用的是数组来进行存储  
- PriorityBlockingQueue 不能存储null 值（队列里面的元素一般具有意义）  
- PriorityBlockingQueue 通过一个重入锁来控制入队和出队操作，线程安全  
- PriorityBlockingQueue 是FIFO队列，但是该FIFO是基于优先级的，通过默认比较器 比较结果中 较小的元素靠近队头（优先级高），
当然我们可以通过自定义比较器来实现排队规则。  
- PriorityBlockingQueue 中没有队满的概念，当元素个数超过队列长度，会进行扩容，当操作队列最大值后（Integer.MAX_VALUE - 8），
将抛出oom异常，同时入队没有队满操作等待和队满阻塞操作，当队列达到最大值，如果继续入队，则会抛oom异常，这一点需要注意，在使用中，
避免大量的元素不断入队，入队速度快，而出队速度又很慢。  
