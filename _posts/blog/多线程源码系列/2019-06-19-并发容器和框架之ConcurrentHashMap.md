---
layout: post
title: 并发容器和框架之ConcurrentHashMap
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---

## 1. 内部类和继承关系
Java8中ConcurrentHashMap增加了很多内部类来支持一些操作和优化性能。下面介绍几个核心的内部类。  
![](/images/posts/多线程/)  

#### 1.1 Node类
存放元素的key,value,hash值,next下一个链表节点的引用。用于bucket为链表时。
#### 1.2 TreeBin
内部属性有root，first节点，以及root节点的锁状态变量lockState，这是一个读写锁的状态。用于存放红黑树的root节点，
并用读写锁lockState控制在写操作即将要调整树结构前，先让读线程完成读操作。从链表结构调整为红黑树时，table中索引下标存储的即为TreeBin。
#### 1.3 TreeNode
红黑树的节点，存放了父节点，左子节点，右子节点的引用，以及红黑节点标识。
#### 1.4 ForwardingNode
在调用transfer()方法期间，插入bucket头部的节点，主要用来标识在扩容时元素的移动状态，即是否在扩容时还有并发的插入节点，
并保证该节点也能够移动到扩容后的表中。
#### 1.5 ReservationNode
占位节点，不存储任何信息，无实际用处，仅用于computeIfAbsent和compute方法中。

## 2. 重要属性介绍

``` java
// table最大容量，为2的幂次方
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认table初始容量大小
    private static final int DEFAULT_CAPACITY = 16;
    // 默认支持并发更新的线程数量
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    // table的负载因子
    private static final float LOAD_FACTOR = 0.75f;
    // 链表转换为红黑树的节点数阈值，超过这个值，链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    // 在扩容期间，由红黑树转换为链表的阈值，小于这个值，resize期间红黑树就会转为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 转为红黑树时，红黑树中节点的最小个数
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 扩容时，并发转移节点(transfer方法)时，每次转移的最小节点数
    private static final int MIN_TRANSFER_STRIDE = 16;

    // 以下常量定义了特定节点类hash字段的值
    static final int MOVED     = -1; // ForwardingNode类对象的hash值
    static final int TREEBIN   = -2; // TreeBin类对象的hash值
    static final int RESERVED  = -3; // ReservationNode类对象的hash值
    static final int HASH_BITS = 0x7fffffff; // 普通Node节点的hash初始值

    // table数组
    transient volatile Node<K,V>[] table;
    // 扩容时，下一个容量大小的talbe，用于将原table元素移动到这个table中
    private transient volatile Node<K,V>[] nextTable;
    // 基础计数器
    private transient volatile long baseCount;
    // table初始容量大小以及扩容容量大小的参数，也用于标识table的状态
    // 其有几个值来代表也用来代表table的状态:
    // -1 ：标识table正在初始化
    // - N : 标识table正在进行扩容，并且有N - 1个线程一起在进行扩容
    // 正数：初始table的大小，如果值大于初始容量大小，则表示扩容后的table大小。
    private transient volatile int sizeCtl;
    // 扩容时，下一个节点转移的bucket索引下标
    private transient volatile int transferIndex;
    // 一种自旋锁，是专为防止多处理器并发而引入的一种锁，用于创建CounterCells时使用，
    // 主要用于size方法计数时，有并发线程插入而计算修改的节点数量，
    // 这个数量会与baseCount计数器汇总后得出size的结果。
    private transient volatile int cellsBusy;
    // 主要用于size方法计数时，有并发线程插入而计算修改的节点数量，
    // 这个数量会与baseCount计数器汇总后得出size的结果。
    private transient volatile CounterCell[] counterCells;
    // 其他省略
```
## 3. 核心方法

#### 3.1 putVal
下面nextTable、sizeCtl、transferIndex与多线程扩容有关，baseCount、cellsBusy、counterCells与新的高效的并发计数方式有关。  
``` java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());// 计算key的hash值
        int binCount = 0;// 表示table中索引下标代表的链表或红黑树中的节点数量
        // 采用自旋方式，等待table第一次put初始化完成，或等待锁或等待扩容成功然后再插入
        for (Node<K,V>[] tab = table;;) {
            // f节点标识table中的索引节点，可能是链表的head，也可能是红黑树的head
            // n:table的长度，i:插入元素在table的索引下标，fh : head节点的hash值
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)// 第一次插入元素，先执行初始化
                tab = initTable();
            // 定位到的索引下标节点(head)为null，表示第一次在此索引插入，
            // 不加锁直接插入在head之后，在casTabAt中采用Unsafe的CAS操作，保证线程安全
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // head节点为ForwadingNode类型节点，表示table正在扩容，链表或红黑树也加入到帮助扩容操作中
            else if ((fh = f.hash) == MOVED) 
                tab = helpTransfer(tab, f);
            else {// 索引下标存在元素，且为普通Node节点，给head加锁后执行插入或更新
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {// 为普通链表节点，还记得之前定义的几种常量Hash值吗？
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 插入新元素，每次插在单向链表的末尾，这点与Java7中不同（插在首部）
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {// head为树节点，按树的方式插入节点
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 链表节点树超过阈值8，将链表转换为红黑树结构
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 如果是插入新元素，则将链表或红黑树最新的节点数量加入到CounterCells中
        addCount(1L, binCount);
        return null;
}
```
总体流程和步骤：  
>1、采用自旋的方式，保证首次put时，当前线程或其他并发put的线程等待table初始化完成后再次重试插入。
 2、采用自旋的方式，检查当前插入的元素在table中索引下标是否正在执行扩容，如果正在扩容，则帮助进行扩容，完成后，重试插入到新的table中。
 3、插入的table索引下标不为空，则对链表或红黑树的head节点加synchronized锁，再插入或更新。访问入口是Head节点，其他线程访问head，
    在链表或红黑树插入或修改时必须等待synchronized释放。
 4、插入后，如果发现链表节点数大于等于阈值8，调用treeifyBin方法，将链表转换为红黑树结构，提高读写性能。
    treeifyBin方法内部也同样采用synchronized方式保证线程安全性。
 5、插入元素后，会将索引代表的链表或红黑树的最新节点数量更新到baseCount或CounterCell中。

##### 3.1.1 spread
计算key的hash值，将key的hashCode的高16位也加入到计算中，避免平凡冲突。如果仅用key的hashCode作为hash值，那么2,4之类的整形key值，
只有低4位，那么很容易发生冲突。
``` java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
##### 3.1.2 initTable
``` java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {// while自旋
        // sizeCtl小于0，表示table正在被其他线程执行初始化，
        // 放弃初始化竞争，自旋等待初始化完成
        // 还记得前面介绍的sizeCtl的含义吗？
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
总体流程和步骤：  
>1、自旋检查table是否完成初始化。
 2、若发现sizeCtl值为负数，则放弃初始化的竞争，让其他正在初始化的线程完成初始化。
 3、如果没有其他线程初始化，则用Unsafe.compareAndSwapInt更新sizeCtl的值为-1，表示table开始被当前线程执行初始化，其他线程禁止执行。
 4、初始化：table设置为默认容量大小（元素并未初始化，只是划定了大小），sizeCtl设为下次扩容table的size大小。
 5、初始化完成。
 
##### 3.1.3 addCount
  
``` java
private final void addCount(long x, int check) {
    // check,即链表或红黑树的节点数，<0不检查是否正在扩容， 
    // <=1仅检查是否存在竞争，没有竞争则直接返回
    CounterCell[] as; long b, s;
    // 如果首次执行addCount，并且尝试用CAS对baseCount计数失败，表示有竞争，则执行如下操作。
    // 或者非首次addCount，也执行如下的操作
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
// sumCount方法
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
总体流程和步骤：  
>1、判断是否首次执行addCount，并判断是否存在竞争关系，如果CAS成功，数量就成功汇总到baseCount中，如果CAS操作失败，则表示有竞争，
有其他线程并发插入，则修改的数量会被记录到CounterCell中。
 2、BaseCount和CounterCell相加就表示正常无并发下的节点数量和并发插入下的节点数量，table索引下标所代表的链表或红黑树节点的数量就能达到精确计算的效果。
 3、在addCount时，还会去检查sizeCtl是否为-N，以确定table是否正在扩容，如果正在扩容，则加入到扩容的操作中。

addCount方法所统计的数值baseCount和counterCells将会被用到size方法中，用于精确计算并发读写情况下table中元素的数量。
#### 3.2 get
get方法步骤：  
1) 计算key的hash值，并定位table索引  
2) 若table索引下元素(head节点)为普通链表，则按链表的形式迭代遍历。  
3) 若table索引下元素为红黑树TreeBin节点，则按红黑树的方式查找(find方法)。  
``` java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {// 普通链表
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // hash值小于-1,即为红黑树，还记得之前定义的TreeBin节点的hash值吗
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {// 匹配下一个链表元素
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
##### 3.2.1 find
步骤如下：  
1) 检查lockState是否为写锁，如果是，则表示有并发写入线程在写入，则按正常的链表方式遍历并查找。  
2) 如果没有写锁，仅加读锁，然后按红黑树的方式查找(TreeBin.findTreeNode方法)。  

``` java
final Node<K,V> find(int h, Object k) {
    if (k != null) {
        for (Node<K,V> e = first; e != null; ) {
            int s; K ek;
            if (((s = lockState) & (WAITER|WRITER)) != 0) {
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                e = e.next;
            }
            else if (U.compareAndSwapInt(this, LOCKSTATE, s, s + READER)) {
                TreeNode<K,V> r, p;
                try {
                    p = ((r = root) == null ? null : r.findTreeNode(h, k, null));
                } finally {
                    Thread w;
                    if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                        (READER|WAITER) && (w = waiter) != null)
                        LockSupport.unpark(w);
                }
                return p;
            }
        }
    }
    return null;
}
```
**疑问解答**：前文不是说了，链表元素超过8个时，会被转成红黑树的结构吗？为什么在树节点遍历方法中，第一点仍然采用链表的方式遍历？  
**回答**：还记得TreeBin和TreeNode节点和Node节点的继承关系吗？Node本身可以链成一个链表，而TreeBin和TreeNode也继承自Node节点，
也自然继承了next属性，同样拥有链表的性质，其实真正在存储时，红黑树仍然是以链表形式存储的，只是逻辑上TreeBin和TreeNode多了支持红黑树的root，
first, parent，left，right，red属性，在附加的属性上进行逻辑上的引用和关联，也就构造成了一颗树。这一点有点像LinkedHashMap，
里面的节点又是在Table中，各个table中的元素又通过before和after引用进行双向链接，达到各个节点之间在逻辑上互链起来的效果。
#### 3.3 transfer
第一部分：
``` java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //计算单个线程允许处理的最少table桶首节点个数，不能小于 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; 
        //刚开始扩容，初始化 nextTab 
        if (nextTab == null) {
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            //transferIndex 指向最后一个桶，方便从后向前遍历 
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //定义 ForwardingNode 用于标记迁移完成的桶
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
```
主要完成的是对单个线程能处理的最少桶结点个数的计算和一些属性的初始化操作。  
<br/>
第二部分：  
``` java
boolean advance = true;
boolean finishing = false;
//i 指向当前桶，bound 指向当前线程需要处理的桶结点的区间下限
for (int i = 0, bound = 0;;) {
       Node<K,V> f; int fh;
       //这个 while 循环的目的就是通过 --i 遍历当前线程所分配到的桶结点
       //一个桶一个桶的处理
       while (advance) {
           int nextIndex, nextBound;
           if (--i >= bound || finishing)
               advance = false;
           //transferIndex <= 0 说明已经没有需要迁移的桶了
           else if ((nextIndex = transferIndex) <= 0) {
               i = -1;
               advance = false;
           }
           //更新 transferIndex
           //为当前线程分配任务，处理的桶结点区间为（nextBound,nextIndex）
           else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,nextBound = 
                                        (nextIndex > stride ? nextIndex - stride : 0))) {
               bound = nextBound;
               i = nextIndex - 1;
               advance = false;
           }
       }
       //当前线程所有任务完成
       if (i < 0 || i >= n || i + n >= nextn) {
           int sc;
           if (finishing) {
               nextTable = null;
               table = nextTab;
               sizeCtl = (n << 1) - (n >>> 1);
               return;
           }
           if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
               if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                   return;
               finishing = advance = true;
               i = n; 
           }
       }
       //待迁移桶为空，那么在此位置 CAS 添加 ForwardingNode 结点标识该桶已经被处理过了
       else if ((f = tabAt(tab, i)) == null)
           advance = casTabAt(tab, i, null, fwd);
       //如果扫描到 ForwardingNode，说明此桶已经被处理过了，跳过即可
       else if ((fh = f.hash) == MOVED)
           advance = true; 
```
每个新参加进来扩容的线程必然先进 while 循环的最后一个判断条件中去领取自己需要迁移的桶的区间。然后 i 指向区间的最后一个位置，
表示迁移操作从后往前的做。接下来的几个判断就是实际的迁移结点操作了。
等我们大致介绍完成第三部分的源码再回来对各个判断条件下的迁移过程进行详细的叙述。  
<br/>
第三部分：  
``` java
else {
    //
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            Node<K,V> ln, hn;
            //链表的迁移操作
            if (fh >= 0) {
                int runBit = fh & n;
                Node<K,V> lastRun = f;
                //整个 for 循环为了找到整个桶中最后连续的 fh & n 不变的结点
                for (Node<K,V> p = f.next; p != null; p = p.next) {
                    int b = p.hash & n;
                    if (b != runBit) {
                        runBit = b;
                        lastRun = p;
                    }
                }
                if (runBit == 0) {
                    ln = lastRun;
                    hn = null;
                }
                else {
                    hn = lastRun;
                    ln = null;
                }
                //如果fh&n不变的链表的runbit都是0，则nextTab[i]内元素ln前逆序，ln及其之后顺序
                //否则，nextTab[i+n]内元素全部相对原table逆序
                //这是通过一个节点一个节点的往nextTab添加
                for (Node<K,V> p = f; p != lastRun; p = p.next) {
                    int ph = p.hash; K pk = p.key; V pv = p.val;
                    if ((ph & n) == 0)
                        ln = new Node<K,V>(ph, pk, pv, ln);
                    else
                        hn = new Node<K,V>(ph, pk, pv, hn);
                }
                //把两条链表整体迁移到nextTab中
                setTabAt(nextTab, i, ln);
                setTabAt(nextTab, i + n, hn);
                //将原桶标识位已经处理
                setTabAt(tab, i, fwd);
                advance = true;
            }
            //红黑树的复制算法，不再赘述
            else if (f instanceof TreeBin) {
                TreeBin<K,V> t = (TreeBin<K,V>)f;
                TreeNode<K,V> lo = null, loTail = null;
                TreeNode<K,V> hi = null, hiTail = null;
                int lc = 0, hc = 0;
                for (Node<K,V> e = t.first; e != null; e = e.next) {
                    int h = e.hash;
                    TreeNode<K,V> p = new TreeNode<K,V>(h, e.key, e.val, null, null);
                    if ((h & n) == 0) {
                        if ((p.prev = loTail) == null)
                            lo = p;
                        else
                            loTail.next = p;
                    loTail = p;
                    ++lc;
                    }
                    else {
                        if ((p.prev = hiTail) == null)
                            hi = p;
                        else
                            hiTail.next = p;
                    hiTail = p;
                    ++hc;
                    }
                }
                ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :(hc != 0) ? new TreeBin<K,V>(lo) : t;
                hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :(lc != 0) ? new TreeBin<K,V>(hi) : t;
                setTabAt(nextTab, i, ln);
                setTabAt(nextTab, i + n, hn);
                setTabAt(tab, i, fwd);
                advance = true;
           }
```
首先，每个线程进来会先领取自己的任务区间，然后开始 --i 来遍历自己的任务区间，对每个桶进行处理。如果遇到桶的头结点是空的，
那么使用 ForwardingNode 标识该桶已经被处理完成了。如果遇到已经处理完成的桶，直接跳过进行下一个桶的处理。如果是正常的桶，对桶首节点加锁，
正常的迁移即可，迁移结束后依然会将原表的该位置标识位已经处理。

















