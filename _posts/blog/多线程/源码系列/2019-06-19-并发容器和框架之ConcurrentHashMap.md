---
layout: post
title: 并发容器和框架之ConcurrentHashMap
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
> 未完成！！！

## 1. JDK8的改动
- 1. jdk1.8的ConcurrentHashMap不再使用Segment代理Map操作这种设计，整体结构变为HashMap这种结构，但是依旧保留分段锁的思想。
之前版本是每个Segment都持有一把锁，1.8版本改为锁住恰好装在一个hash桶本身位置上的节点，也就是hash桶的第一个节点 tabAt(table, i)，
后面直接叫第一个节点。它可能是Node链表的头结点、保留节点ReservationNode、或者是TreeBin节点（TreeBin节点持有红黑树的根节点）。
还有，1.8的节点变成了4种，这个后面细说。  
- 2. 可以多线程并发来完成扩容这个耗时耗力的操作。在之前的版本中如果Segment正在进行扩容操作，其他写线程都会被阻塞，
jdk1.8改为一个写线程触发了扩容操作，其他写线程进行写入操作时，可以帮助它来完成扩容这个耗时的操作。多线程并发扩容这部分后面细说。  
- 3. 因为多线程并发扩容的存在，导致的其他操作的实现上会有比较大的改动，常见的get/put/remove/replace/clear，以及迭代操作，
都要考虑并发扩容的影响。  
- 4. 使用新的计数方法。不使用Segment时，如果直接使用一个volatile类变量计数，因为每次读写volatile变量的开销很大，
高并发时效率不如之前版本的使用Segment时的计数方式。
jdk1.8新增了一个用与高并发情况的计数工具类java.util.concurrent.atomic.LongAdder，
此类是基本思想和1.7及以前的ConcurrentHashMap一样，使用了一层中间类，叫做Cell（类似Segment这个类）的计数单元，来实现分段计数，
最后合并统计一次。因为不同的计数单元可以承担不同的线程的计数要求，减少了线程之间的竞争，在1.8的ConcurrentHashMap基本结果改变时，
继续保持和分段计数一样的并发计数效率。  
5. 同1.8版本的HashMap，当一个hash桶中的hash冲突节点太多时，把链表变为红黑树，提高冲突时的查找效率。  
6. 一些小的改进，具体见后面的源码上写的注释。  

## 2. 常量和变量
#### 2.1 常量

``` java
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final int DEFAULT_CAPACITY = 16;
 
// 下面3个，在1.8的HashMap中也有相同的常量
 
// 一个hash桶中hash冲突的数目大于此值时，把链表转化为红黑树，加快hash冲突时的查找速度
static final int TREEIFY_THRESHOLD = 8;
// 一个hash桶中hash冲突的数目小于等于此值时，把红黑树转化为链表，当数目比较少时，
// 链表的实际查找速度更快，也是为了查找效率
static final int UNTREEIFY_THRESHOLD = 6;
// 当table数组的长度小于此值时，不会把链表转化为红黑树。所以转化为红黑树有两个条件，
//还有一个是 TREEIFY_THRESHOLD
static final int MIN_TREEIFY_CAPACITY = 64;
 
// 虚拟机限制的最大数组长度，jdk1.8新引入的，ConcurrentHashMap的主体代码中是不使用这个的，
// 主要用在Collection.toArray两个方法中
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
 
// 默认并行级别，主体代码中未使用此常量，主要是为了兼容Segment使用，另外在构造方法中有一些作用
// 千万注意，1.8的并发级别有了大的改动，具体并发级别可以认为是hash桶的数量，也就是容量，
// 会随扩容而改变，不再是固定值
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
 
// 加载因子，为了兼容性，保留了这个常量（名字变了），配合同样是为了兼容性的Segment使用
// 1.8的ConcurrentHashMap的加载因子固定为 0.75，构造方法中指定的参数是不会被用作loadFactor的，
// 为了计算方便，统一使用 n - (n >> 2) 代替浮点乘法 *0.75
private static final float LOAD_FACTOR = 0.75f;
 
// 扩容操作中，transfer这个步骤是允许多线程的，这个常量表示一个线程执行transfer时，
// 最少要对连续的16个hash桶进行transfer（不足16就按16算，多控制下正负号就行）

// 也就是单线程执行transfer时的最小任务量，单位为一个hash桶，这就是线程的transfer的步进（stride）
// 最小值是DEFAULT_CAPACITY，不使用太小的值，避免太小的值引起transfer时线程竞争过多，
// 如果计算出来的值小于此值，就使用此值

// 正常步骤中会根据CPU核心数目来算出实际的，一个核心允许8个线程并发执行扩容操作的transfer步骤，
// 这个8是个经验值，不能调整的。因为transfer操作不是IO操作，也不是死循环那种100%的CPU计算，CPU计算率中等，
// 1核心允许8个线程并发完成扩容，理想情况下也算是比较合理的值。

// 一段代码的IO操作越多，1核心对应的线程就要相应设置多点;CPU计算越多，1核心对应的线程就要相应设置少一些
// 表明：默认的容量是16，也就是默认构造的实例，第一次扩容实际上是单线程执行的，看上去是可以多线程并发
//（方法允许多个线程进入），但是实际上其余的线程都会被一些if判断拦截掉，不会真正去执行扩容
private static final int MIN_TRANSFER_STRIDE = 16;
 
// 用于每次扩容生成唯一的生成戳的数，最小是6。很奇怪，这个值不是常量，但是也不提供修改方法。
private static int RESIZE_STAMP_BITS = 16;
 
// 最大的扩容线程的数量，如果上面的 RESIZE_STAMP_BITS = 32，那么此值为 0，这一点也很奇怪。
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
 
// 移位量，把生成戳移位后保存在sizeCtl中当做扩容线程计数的基数，相反方向移位后能够反解出生成戳
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
 
// 下面几个是特殊的节点的hash值，正常节点的hash值在hash函数中都处理过了，不会出现负数的情况，
// 特殊节点在各自的实现类中有特殊的遍历方法


// ForwardingNode的hash值，ForwardingNode是一种临时节点，在扩进行中才会出现，并且它不存储实际的数据
// 如果旧数组的一个hash桶中全部的节点都迁移到新数组中，旧数组就在这个hash桶中放置一个ForwardingNode
// 读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，
// 写操作碰见它时，则尝试帮助扩容
static final int MOVED     = -1; // hash for forwarding nodes
 
// TreeBin的hash值，TreeBin是ConcurrentHashMap中用于代理操作TreeNode的特殊节点，
// 持有存储实际数据的红黑树的根节点。因为红黑树进行写入操作，整个树的结构可能会有很大的变化，
// 这个对读线程有很大的影响，所以TreeBin还要维护一个简单读写锁，这是相对HashMap新引入这特殊节点的重要原因
static final int TREEBIN   = -2; // hash for roots of trees
 
// ReservationNode的hash值，ReservationNode是一个保留节点，就是个占位符，不会保存实际的数据，
// 正常情况是不会出现的，在jdk1.8新的函数式有关的两个方法computeIfAbsent和compute中才会出现
static final int RESERVED  = -3; // hash for transient reservations
 
// 用于和负数hash值进行 & 运算，将其转化为正数（绝对值不相等），
// Hashtable中定位hash桶也有使用这种方式来进行负数转正数
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
 
// CPU的核心数，用于在扩容时计算一个线程一次要干多少活
static final int NCPU = Runtime.getRuntime().availableProcessors();
 
// 在序列化时使用，这是为了兼容以前的版本
private static final ObjectStreamField[] serialPersistentFields = {
    new ObjectStreamField("segments", Segment[].class),
    new ObjectStreamField("segmentMask", Integer.TYPE),
    new ObjectStreamField("segmentShift", Integer.TYPE)
};
 
// Unsafe初始化跟1.7版本的基本一样，不说了
```
#### 2.2 变量
下面nextTable、sizeCtl、transferIndex与多线程扩容有关，baseCount、cellsBusy、counterCells与新的高效的并发计数方式有关。  
``` java
transient volatile Node<K,V>[] table;
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
 
// 即扩容后的新的table数组。这个变量只有在扩容时才有用。nextTable != null，说明扩容方法还没有真正退出，
// 一般可以认为是此时还有线程正在进行扩容，极端情况需要考虑此时扩容操作只差最后给几个变量赋值
//（包括nextTable = null）的这个大的步骤，这个大步骤执行时，sizeCtl经过一些计算得出来的扩容线程的数量是0
private transient volatile Node<K,V>[] nextTable;
 
// 非常重要的一个属性，源码中的英文翻译，直译过来是下面的四行文字的意思
//     sizeCtl = -1，表示有线程正在进行真正的初始化操作
//     sizeCtl = -(1 + nThreads)，表示有nThreads个线程正在进行扩容操作
//     sizeCtl > 0，表示接下来的真正的初始化操作中使用的容量，或者初始化/扩容完成后的threshold
//     sizeCtl = 0，默认值，此时在真正的初始化操作中使用默认容量
// 有问题的是第二句，sizeCtl = -(1 + nThreads)这个，默认构造的16个大小的ConcurrentHashMap，
// 当只有一个线程执行扩容时，sizeCtl = -2145714174，但照英文注释的意思，sizeCtl的值应该是 -(1 + 1) = -2
// sizeCtl在小于0时的确有记录有多少个线程正在执行扩容任务的功能，但不是英文注释说的直接用 -(1 + nThreads)
// 实际中使用一种生成戳，根据生成戳算出一个基数，不同轮次的扩容的生成戳都是唯一的，保证多次扩容间不交叉重叠，
// 当有n个线程正在执行扩容时，sizeCtl在值变为 (基数 + n)
private transient volatile int sizeCtl;
 
// 即下一个transfer任务的起始下标index 加上1 之后的值。
// transfer时下标index从length-1开始往0走，迭代时是下标从小往大，二者方向相反，
// 尽量减少扩容时的transefer和迭代两者同时处理一个hash桶的情况。顺序相反时，二者相遇过后，
// 迭代没处理的都是已经transfer的hash桶，transfer没处理的，都是已经迭代的hash桶，冲突会变少。
// 下标在[nextIndex - 实际的stride （下界要 >= 0）, nextIndex - 1]内的hash桶，就是每个transfer的任务区间
// 每次接受一个transfer任务，都要CAS执行 transferIndex = transferIndex - 实际的stride，
// 保证一个transfer任务不会被几个线程同时获取（相当于任务队列的size减1）
// 当没有线程正在执行transfer任务时，一定有transferIndex <= 0，这是判断是否需要帮助扩容的重要条件
//（相当于任务队列为空）
private transient volatile int transferIndex;
 
// 下面三个主要与统计数目有关，主要在没有碰到多线程竞争时使用，需要通过CAS进行更新
private transient volatile long baseCount;
 
// CAS自旋锁标志位，用于初始化，或者counterCells扩容时
private transient volatile int cellsBusy;
 
// 用于高并发的计数单元，如果初始化了这些计数单元，那么跟table数组一样，长度必须是2^n的形式
private transient volatile CounterCell[] counterCells;
```
## 3. 基本类
#### 3.1 Node：基本节点/普通节点
此节点就是一个很普通的Entry，在链表形式保存才使用这种节点，它存储实际的数据，基本结构类似于1.8的HashMap.Node。  
``` java
// 此类不会在ConcurrentHashMap以外被修改，只读迭代可利用这个类，
// 迭代时的写操作需要由另一个内部类MapEntry代理执行写操作。
// 此类的子类具有负数hash值，并且不存储实际的数据，如果不使用子类直接使用这个类，那么key和val永远不为null
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
 
    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }
 
    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    
    // 不支持来自ConcurrentHashMap外部的修改，迭代操作需要通过另外一个内部类MapEntry来代理，
    // 迭代写会重新执行一次put操作，迭代中可以改变value，是一种写操作，此时需要保证这个节点还在map中，
    // 因此就重新put一次：节点不存在了，可以重新让它存在；节点还存在，相当于replace一次
    // 设计成这样主要是因为ConcurrentHashMap并非为了迭代操作而设计，它的迭代操作和其他写操作不好并发，
    // 迭代时的读写都是弱一致性的，碰见并发修改时尽量维护迭代的一致性
    // 返回值V也可能是个过时的值，保证V是最新的值会比较困难，而且得不偿失
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }
 
    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&  (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&  
                    (v = e.getValue()) != null && (k == key || k.equals(key)) &&  
                        (v == (u = val) || v.equals(u))); 
    }
 
    // 从此节点开始查找k对应的节点。这里的实现是专为链表实现的，一般作用于头结点，
    // 各种特殊的子类有自己独特的实现。不过主体代码中进行链表查找时，因为要特殊判断下第一个节点，
    // 所以很少直接用下面这个方法，而是直接写循环遍历链表，子类的查找则是用子类中重写的find方法
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&  ((ek = e.key) == k || (ek != null && k.equals(ek)))) 
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```
#### 3.2 TreeNode：红黑树节点
在红黑树形式保存时才存在，它也存储有实际的数据，结构和1.8的HashMap的内部类TreeNode一样，一些方法的实现代码也基本一样。不过，
ConcurrentHashMap对此节点的操作，都会由TreeBin来代理执行。也可以把这里的TreeNode看出是有一半功能的HashMap.TreeNode，
另一半功能在ConcurrentHashMap.TreeBin中。  
<br/>
红黑树节点本身保存有普通链表节点Node的所有属性，因此可以使用两种方式进行读操作。  
``` java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    // 新添加的prev指针是为了删除方便，删除链表的非头节点的节点，
    // 都需要知道它的前一个节点才能进行删除，所以直接提供一个prev指针
    TreeNode<K,V> prev; 
    boolean red;
 
    TreeNode(int hash, K key, V val, Node<K,V> next, TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }
 
    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }
 
    // 以当前节点 this 为根节点开始遍历查找，跟HashMap.TreeNode.find实现一样
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        if (k != null) {
            TreeNode<K,V> p = this;
            do  {
                int ph, dir; K pk; TreeNode<K,V> q;
                TreeNode<K,V> pl = p.left, pr = p.right;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null || (kc = comparableClassFor(k)) != null) && 
                            (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.findTreeNode(h, k, kc)) != null) // 对右子树进行递归查找
                    return q;
                else
                    p = pl; // 前面递归查找了右边子树，这里循环时只用一直往左边找
            } while (p != null);
        }
        return null;
    }
}
```
#### 3.3 ForwardingNode：转发节点
ForwardingNode是一种临时节点，在扩容进行中才会出现，hash值固定为-1，并且它不存储实际的数据数据。
如果旧数组的一个hash桶中全部的节点都迁移到新数组中，旧数组就在这个hash桶中放置一个ForwardingNode。
读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容。
``` java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
 
    // ForwardingNode的查找操作，直接在新数组nextTable上去进行查找
    Node<K,V> find(int h, Object k) {
        // 使用循环，避免多次碰到ForwardingNode导致递归过深
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||  
                    (e = tabAt(tab, (n - 1) & h)) == null) 
                return null;
            for (;;) {
                int eh; K ek;
                // 第一个节点就是要找的节点，直接返回
                if ((eh = e.hash) == h &&  ((ek = e.key) == k || (ek != null && k.equals(ek)))) 
                    return e;
                if (eh < 0) {
                    // 继续碰见ForwardingNode的情况，这里相当于是递归调用一次本方法
                    if (e instanceof ForwardingNode) { 
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k); // 碰见特殊节点，调用其find方法进行查找
                }
                if ((e = e.next) == null) // 普通节点直接循环遍历链表
                    return null;
            }
        }
    }
}
```
#### 3.4 TreeBin：代理操作TreeNode的节点
TreeBin的hash值固定为-2，它是ConcurrentHashMap中用于代理操作TreeNode的特殊节点，持有存储实际数据的红黑树的根节点。
因为红黑树进行写入操作，整个树的结构可能会有很大的变化，这个对读线程有很大的影响，所以TreeBin还要维护一个简单读写锁，
这是相对HashMap，这个类新引入这种特殊节点的重要原因。  
``` java
// 红黑树节点TreeNode实际上还保存有链表的指针，因此也可以用链表的方式进行遍历读取操作
// 自身维护一个简单的读写锁，不用考虑写-写竞争的情况。
// 不是全部的写操作都要加写锁，只有部分的put/remove需要加写锁
// 很多方法的实现和jdk1.8的ConcurrentHashMap.TreeNode里面的方法基本一样，可以互相参考
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root; // 红黑树结构的跟节点
    volatile TreeNode<K,V> first; // 链表结构的头节点
    volatile Thread waiter; // 最近的一个设置 WAITER 标识位的线程
    volatile int lockState; // 整体的锁状态标识位
 
    // 二进制001，红黑树的 写锁状态
    static final int WRITER = 1; 
    // 二进制010，红黑树的 等待获取写锁的状态，中文名字太长，后面用 WAITER 代替
    static final int WAITER = 2; // 
    // 二进制100，红黑树的 读锁状态，读锁可以叠加，
    // 也就是红黑树方式可以并发读，每有一个这样的读线程，lockState都加上一个READER的值
    static final int READER = 4; 
 
    // 重要的一点，红黑树的 读锁状态 和 写锁状态 是互斥的，
    // 但是从ConcurrentHashMap角度来说，读写操作实际上可以是不互斥的
    //
    // 红黑树的 读、写锁状态 是互斥的，指的是以红黑树方式进行的读操作和写操作
    // （只有部分的put/remove需要加写锁）是互斥的
    // 
    // 但是当有线程持有红黑树的 写锁 时，读线程不会以红黑树方式进行读取操作，
    // 而是使用简单的链表方式进行读取，此时读操作和写操作可以并发执行  
    // 
    // 当有线程持有红黑树的 读锁 时，写线程可能会阻塞，不过因为红黑树的查找很快，写线程阻塞的时间很短
    //
    // 另外一点，ConcurrentHashMap的put/remove/replace方法本身就会锁住TreeBin节点，
    // 这里不会出现写-写竞争的情况，因此这里的读写锁可以实现得很简单
 
    // 在hashCode相等并且不是Comparable类时才使用此方法进行判断大小
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null || (d = a.getClass().getName().compareTo(b.getClass().getName())) == 0)
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ? -1 : 1);
        return d;
    }
 
    // 用以b为头结点的链表创建一棵红黑树
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K,V> r = null;
        for (TreeNode<K,V> x = b, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = r;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null && (kc = comparableClassFor(k)) == null) || 
                                (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                        TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }
 
    
    // 对根节点加 写锁，红黑树重构时需要加上 写锁
    private final void lockRoot() {
        if (!U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER)) // 先尝试获取一次 写锁
            contendedLock(); // 单独抽象出一个方法，直到获取到 写锁 这个调用才会返回
    }
 
    // 释放 写锁
    private final void unlockRoot() {
        lockState = 0;
    }
 
    // 可能会阻塞写线程，当写线程获取到写锁时，才会返回
    // ConcurrentHashMap的put/remove/replace方法本身就会锁住TreeBin节点，这里不会出现写-写竞争的情况
    // 本身这个方法就是给写线程用的，因此只用考虑 读锁 阻碍线程获取 写锁，不用考虑 写锁 阻碍线程获取 写锁，
    // 这个读写锁本身实现得很简单，处理不了写-写竞争的情况
    // waiter要么是null，要么是当前线程本身
    private final void contendedLock() {
        boolean waiting = false;
        for (int s;;) {
            //下面都是以没有获取到写锁为前提，因为能进入到这个方法说明没有获取到写锁
        
            // ~WAITER是对WAITER进行二进制取反。当此时没有线程持有 读锁时，这个if为真
            if (((s = lockState) & ~WAITER) == 0) {
                // 在 读锁、写锁 都没有被别的线程持有时，尝试为自己这个写线程获取 写锁
                if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                    
                    if (waiting) // 获取到写锁时，如果自己曾经注册过 WAITER 状态，将其清除
                        waiter = null;
                    return;
                }
            }
            
            // 有线程持有 读锁 ，并且当前线程不是 WAITER 状态时，这个else if为真
            else if ((s & WAITER) == 0) { 
                if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) { //尝试占据 WAITER 状态标识位
                    waiting = true; // 表明自己正处于 WAITER 状态，并且让下一个被用于进入下一个 else if
                    waiter = Thread.currentThread();
                }
            }
            else if (waiting) // 有线程持有 读锁，并且有线程处于 WAITER 状态时，这个else if为真
                LockSupport.park(this); // 阻塞自己
        }
    }
 
    // 从根节点开始遍历查找，找到“相等”的节点就返回它，没找到就返回null
    // 当有写线程加上 写锁 时，使用链表方式进行查找
    final Node<K,V> find(int h, Object k) {
        if (k != null) {
            for (Node<K,V> e = first; e != null; ) {
                int s; K ek;
                // 两种特殊情况下以链表的方式进行查找
                // 1、有线程正持有 写锁，这样做能够不阻塞读线程
                // 2、WAITER时，不再继续加 读锁，能够让已经被阻塞的写线程尽快恢复运行，
                // 或者刚好让某个写线程不被阻塞
                if (((s = lockState) & (WAITER|WRITER)) != 0) {
                    if (e.hash == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    e = e.next;
                }
                // 读线程数量加1，读状态进行累加
                else if (U.compareAndSwapInt(this, LOCKSTATE, s, s + READER)) { 
                    TreeNode<K,V> r, p;
                    try {
                        p = ((r = root) == null ? null : r.findTreeNode(h, k, null));
                    } finally {
                        Thread w;
                        // 如果这是最后一个读线程，并且有写线程因为 读锁 而阻塞，那么要通知它，
                        // 告诉它可以尝试获取写锁了
                        // U.getAndAddInt(this, LOCKSTATE, -READER)这个操作是在更新之后返回lockstate的旧值，
                        // 不是返回新值，相当于先判断==，再执行减法
                        if (U.getAndAddInt(this, LOCKSTATE, -READER) == (READER|WAITER) && 
                            (w = waiter) != null)
                            LockSupport.unpark(w); // 让被阻塞的写线程运行起来，重新去尝试获取 写锁
                    }
                    return p;
                }
            }
        }
        return null;
    }
 
    // 用于实现ConcurrentHashMap.putVal
    final TreeNode<K,V> putTreeVal(int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if (p == null) {
                first = root = new TreeNode<K,V>(h, k, v, null, null);
                break;
            }
            else if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                return p;
            else if ((kc == null && (kc = comparableClassFor(k)) == null) || 
                    (dir = compareComparables(kc, k, pk)) == 0) {
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null && (q = ch.findTreeNode(h, k, kc)) != null) ||
                        ((ch = p.right) != null && (q = ch.findTreeNode(h, k, kc)) != null))
                        return q;
                }
                dir = tieBreakOrder(k, pk);
            }
 
            TreeNode<K,V> xp = p;
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                TreeNode<K,V> x, f = first;
                first = x = new TreeNode<K,V>(h, k, v, f, xp);
                if (f != null)
                    f.prev = x;
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                // 下面是有关put加 写锁 部分
                // 二叉搜索树新添加的节点，都是取代原来某个的NIL节点（空节点，null节点）的位置
                
                // xp是新添加的节点的父节点，如果它是黑色的，
                // 新添加一个红色节点就能够保证x这部分的一部分路径关系不变，
                if (!xp.red) 
                    // 因为这种情况就是在树的某个末端添加节点，不会改变树的整体结构，
                    // 对读线程使用红黑树搜索的搜索路径没影响
                    x.red = true; 
                // 其他情况下会有树的旋转的情况出现，当读线程使用红黑树方式进行查找时，
                // 可能会因为树的旋转，导致多遍历、少遍历节点，影响find的结果
                else { 
                    lockRoot(); // 除了那种最最简单的情况，其余的都要加写锁，让读线程用链表方式进行遍历读取
                    try {
                        root = balanceInsertion(root, x);
                    } finally {
                        unlockRoot();
                    }
                }
                break;
            }
        }
        assert checkInvariants(root);
        return null;
    }
 
    // 基本是同jdk1.8的HashMap.TreeNode.removeTreeNode，仍然是从链表以及红黑树上都删除节点
    // 两点区别：1、返回值，红黑树的规模太小时，返回true，调用者再去进行树->链表的转化；
    //           2、红黑树规模足够，不用变换成链表时，进行红黑树上的删除要加 写锁
    final boolean removeTreeNode(TreeNode<K,V> p) {
        TreeNode<K,V> next = (TreeNode<K,V>)p.next;
        TreeNode<K,V> pred = p.prev;  // unlink traversal pointers
        TreeNode<K,V> r, rl;
        if (pred == null)
            first = next;
        else
            pred.next = next;
        if (next != null)
            next.prev = pred;
        if (first == null) {
            root = null;
            return true;
        }
        if ((r = root) == null || r.right == null || (rl = r.left) == null || rl.left == null) // too small
            return true;
        lockRoot();
        try {
            TreeNode<K,V> replacement;
            TreeNode<K,V> pl = p.left;
            TreeNode<K,V> pr = p.right;
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
                    p.parent = s;
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s;
                if ((s.parent = pp) == null)
                    r = s;
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    r = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }
 
            root = (p.red) ? r : balanceDeletion(r, replacement);
 
            if (p == replacement) {  // detach pointers
                TreeNode<K,V> pp;
                if ((pp = p.parent) != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                    p.parent = null;
                }
            }
        } finally {
            unlockRoot();
        }
        assert checkInvariants(root);
        return false;
    }
 
    // 下面四个是经典的红黑树方法，改编自《算法导论》
    static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root, TreeNode<K,V> p);
    static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root, TreeNode<K,V> p);
    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x);
    static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x);
    // 递归检查一些关系，确保构造的是正确无误的红黑树
    static <K,V> boolean checkInvariants(TreeNode<K,V> t);
    // Unsafe相关的初始化工作
    private static final sun.misc.Unsafe U;
    private static final long LOCKSTATE;
    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = TreeBin.class;
            LOCKSTATE = U.objectFieldOffset(k.getDeclaredField("lockState"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```
#### 3.5 ReservationNode：保留节点
或者叫空节点，computeIfAbsent和compute这两个函数式api中才会使用。它的hash值固定为-3，就是个占位符，不会保存实际的数据，
正常情况是不会出现的，在jdk1.8新的函数式有关的两个方法computeIfAbsent和compute中才会出现。  
<br/>
为什么需要这个节点，因为正常的写操作，都会想对hash桶的第一个节点进行加锁，但是null是不能加锁，所以就要new一个占位符出来，
放在这个空hash桶中成为第一个节点，把占位符当锁的对象，这样就能对整个hash桶加锁了。
put/remove不使用ReservationNode是因为它们都特殊处理了下，并且这种特殊情况实际上还更简单，put直接使用cas操作，remove直接不操作，
都不用加锁。但是computeIfAbsent和compute这个两个方法在碰见这种特殊情况时稍微复杂些，代码多一些，不加锁不好处理，
所以需要ReservationNode来帮助完成对hash桶的加锁操作。  
``` java
static final class ReservationNode<K,V> extends Node<K,V> {
    ReservationNode() {
        super(RESERVED, null, null, null);
    }
 
    // 空节点代表这个hash桶当前为null，所以肯定找不到“相等”的节点
    Node<K,V> find(int h, Object k) {
        return null;
    }
}
```
## 4. 构造方法与初始化
下面是构造方法，不执行真正的初始化。
``` java
// 什么也不做
public ConcurrentHashMap() {
}
 
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1)); // 求 2^n
    this.sizeCtl = cap;  // 用这个重要的变量保存hash桶的接下来的初始化使用的容量
}
 
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
 
// concurrencyLevel只是为了此方法能够兼容之前的版本，它并不是实际的并发级别，loadFactor也不是实际的加载因子了
// 这两个都失去了原有的意义，仅仅对初始容量有一定的控制作用
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0) // 检查参数
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)
        initialCapacity = concurrencyLevel;
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size); // tableSizeFor，求不小于size的 2^n的算法
    this.sizeCtl = cap; // 用这个重要的变量保存hash桶的接下来的初始化使用的容量
    // 不进行任何数组（hash桶）的初始化工作，构造方法进行懒初始化
}
 
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
```
真正的初始化在iniTable()方法中，在put方法中有调用此方法
``` java
// 真正的初始化方法，使用保存在sizeCtl中的数据作为初始化容量
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // Thread.yeild() 和 CAS 都不是100%和预期一致的方法，所以用循环，其他代码中也有很多这样的场景
    while ((tab = table) == null || tab.length == 0) { 
        if ((sc = sizeCtl) < 0) // 看前面sizeCtl这个重要变量的注释
            // 真正的初始化是要禁止并发的，保证tables数组只被初始化一次，但是又不能切换线程，
            // 所以用yeild()暂时让出CPU
            Thread.yield(); 
                            
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // CAS更新sizeCtl标识为 "初始化" 状态
            try {
                // 检查table数组是否已经被初始化，没初始化就真正初始化
                if ((tab = table) == null || tab.length == 0) { 
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // sc = threshold，n - (n >>> 2) = n - n/4 = 0.75n，
                    // 前面说了loadFactor没用了，这里看出，统一用0.75n了
                    sc = n - (n >>> 2); 
                }
            } finally {
                sizeCtl = sc; // 设置threshold
            }
            break;
        }
    }
    return tab;
}
```
## 5. 一些基本的方法
基本还是和1.8的HashMap，以及1.7的ConcurrentHashMap中的那些对应的基本方法差不多。
``` java
// 扰乱key的hash code，即把高16位与低16位进行异或运算，
// 然后将扰乱后的hash code与实际可访问到的最大索引进行与运算，得出的结果即是目标所在的位置。  
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;//HASH_BITS = 0x7fffffff; 用来保证最后hash永远为正数
}
 
// 用于求2^n，用来作为table数组的容量，同1.8的HashMap
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
 
// 用于获取Comparable接口中的泛型类
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) {
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() == Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}
 
// 当类型相同且实现Comparable时，调用compareTo比较大小
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :  ((Comparable)k).compareTo(x)); 
}
 
// 下面几个用于读写table数组，使用Unsafe提供的更强的功能（数组元素的volatile读写，CAS 更新）
// 代替普通的读写，调用者预先进行参数控制
 
// volatile读取table[i]
@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
 
// CAS更新table[i]，也就是Node链表的头节点，或者TreeBin节点（它持有红黑树的根节点）
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
 
// volatile写入table[i]
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
 
// 满足变换为红黑树的两个条件时（链表长度这个条件调用者保证，这里只验证Map容量这个条件），
// 将链表变为红黑树，否则只是进行一次扩容操作
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) // Map的容量不够时，只是进行一次扩容
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p = new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
 
// 规模不足时把红黑树转化为链表，此方法由调用者进行synchronized加锁，所以这里不加锁
static <K,V> Node<K,V> untreeify(Node<K,V> b) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = b; q != null; q = q.next) {
        Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```
## 6. 计数操作
1.7及以前的ConcurrentHashMap中使用了Segment，Segment能够分担所有的针对单个K-V的写操作，包括put/replace。
并且Segment自带一些数据，比如Segment.count，用于处理Map的计数要求，这样就可以像put/repalce一样，分担整个Map的并发计数压力。  
<br/>
但是1.8中没有再使用Segment来完成put/replace，虽然还是利用了锁分段的思想，
但是使用的是自带的synchronized锁住hash桶中的第一个节点，没有新增别的数据。因此计数操作，被落下来了，
它无法享受synchronized实现的变种分段锁带来的高效率，单独使用一个Map.size来计数，线程竞争可能会很大，比使用Segment是效率低很多。  
<br/>
为了处理这个问题，jdk1.8中使用了一个仿造LongAdder实现的计数器，让计数操作额外使用别的基于分段并发思想的实现的类。
ConcurrentHashMap中不直接使用LongAdder，而是自己拷贝代码实现一个内部的，主要为了方便。LongAdder的实现本身代码不是特别多，
ConcurrentHashMap中的实现，基本和LongAdder一样，可以直接看做是LongAdder。  
## 7. 扩容
#### 7.1 一个transfer任务
对于一个大任务拆分成多个小任务供多线程执行，一般都要求这些小任务具有相似性，流程一致，并且很重要的一点，
任务之间的相互影响尽量少。那么在扩容之中，是怎么划分这个任务的呢？  
<br/>
一般我们说的扩容，都包含两个步骤。第一，新建一个2倍大小的数组，这个过程要求单线程完成（多线程创建几个数组没有意义，容易出错）。
第二步，执行节点迁移，说白了就是rehash，相当于把旧数组中所有的节点重新“put”到新数组中。在1.8的HashMap中，用了一个技巧，
避免了重新根据hash值定位。根据这个，我们可以知道，进行 n -> 2n 的扩容时，扩容前节点所在的hash桶的索引为index，
这个节点迁移到新数组中只会有两种情况：要么在还是在新数组的索引为index处，要么迁移到新数组的索引为 index + n 的地方。
所以旧的table数组上各个hash桶中的节点的迁移是不会互相影响的，这一点对多线程扩容非常有利。根据这一点，可以知道，
每个hash桶的迁移都可以作为一个线程在扩容时的一个transfer任务。  
<br/>
另外，每个线程要求任务都不应该规模太小，因为扩容并不是IO型操作，节点迁移的执行速度本身很快，太多的线程来执行节点迁移，
线程调度开销占比变大，反而降低了吞吐量。ConcurrentHashMap这里，会根据CPU的核心数目，来算出一个transfer任务包含的hash桶的数量。  
``` java
// 下面的代码用于计算每个transfer中要迁移多少个hash桶，一个transfer任务完成后，可以再次申请
int n = tab.length, stride;
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
    stride = MIN_TRANSFER_STRIDE; 
```
为了好说明，下面统一使用最小值MIN_TRANSFER_STRIDE，16，也就是1个线程的一次transfer任务要负责迁移16个hash桶。
#### 7.2 transfer任务的申请
这个就是讲解transferIndex这个属性的作用。
在经典的生成者消费者模式中，会有一个任务队列。消费者向任务队列申请任务，申请到任务，队列中元素数减1，当任务队列中元素数量为0时，
不能再申请任务。ConcurrentHashMap这里并没有使用额外的任务队列，因为table数组本身就可以当作是一个队列。
第一点中说了，一个transfer任务中要负责迁移stride个hash桶，最简单的设计当然是16个hash桶都是连续的。为了记录还有多少个任务，
使用了一个类变量transferIndex，可以把这个看成是任务队列的size，每一次申请任务，这个size减1。  
<br/>
另外，迭代操作的下标是从小往大，也就是正向，为了减少扩容时的transfer和迭代的冲突，transfer使用反向，也就是下标从大到小。
顺序相反时，二者相遇过后，迭代没处理的都是已经transfer的hash桶，transfer没处理的，都是已经迭代的hash桶，冲突会变少。  
<br/>
所以，任务队列的size减1，翻译过来就是，在table数组中的索引减stride。这个索引就是 transferIndex，
用于标记整体的transfer进行到了哪里。因为transfer个数，从1开始，因此transferIndex也是从1开始，
下标在[transferIndex - 实际的stride, transferIndex - 1]内的hash桶，就是每个transfer的任务区间。
transferIndex <= 0 时，代表没有任务可以申请，此时无法帮助扩容。注意，NCPU不一定是2^n，
因此最后一个任务中的hash桶的数量可能不足stride个，此时只执行余下的数量。为了保证每个任务只被领取一次，
transferIndex递减是用CAS操作完成的。  
<br/>
特殊情况下，会出现多线程扩容重叠，此时某个transfer任务虽然被领取了，但是却不能被执行，会被作废。
这是根据transfer方法的代码理解出来的的，因为transfer方法的代码中有考虑任务作废的情况。但是根据下面第3点的分析，
扩容重叠这种特殊情况是有一个机制来避免的。根据本人目前对代码的理解， 简而言之就是：代码中有处理扩容作废，但是实际不会发生。
![](/images/posts/多线程/)  
#### 7.3 resizeStamp以及扩容重叠相关
``` java
// 用于每次扩容生成唯一的生成戳的数，最小是6。很奇怪，这个值不是常量，但是也不提供修改方法。
private static int RESIZE_STAMP_BITS = 16;
 
// 最大的扩容线程的数量，如果上面的 RESIZE_STAMP_BITS = 32，那么此值为 0，这一点也很奇怪。
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
 
// 移位量，把生成戳移位后保存在sizeCtl中当做扩容线程计数的基数，相反方向移位后能够反解出生成戳
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
 
//见上面
private transient volatile int sizeCtl;
 
// 返回与扩容有关的一个生成戳rs，每次新的扩容，都有一个不同的n，这个生成戳就是根据n来计算出来的一个数字，
// n不同，这个数字也不同。另外还得保证 rs << RESIZE_STAMP_SHIFT 必须是负数
// 这个方法的返回值，当且仅当 RESIZE_STAMP_SIZE = 32时为负数
// 但是b = 32时MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1 = 0，这一点很奇怪
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
 
// 这个if在addCount、helpTransfer、tryPresize中都有（可能会少一个条件，因为那个条件上文判断了）
// 实际看下代码，可以知道执行到这里时，sc（sc = sizeCtl）是一定小于0的
// 为真时会直接退出外层循环，然后退出方法
if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS 
    || (nt = nextTable) == null || transferIndex <= 0)
 
// 这个在addCount、tryPresize中都有。实际看下代码，可以知道执行到这里时，
// sc 一定大于 0（等于0是初始化情况，可以不考虑）为真时会进入transfer方法去执行扩容操作
else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
 
// 这个在transfer方法中，条件为真时会 return 退出方法
if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)先看下resizeStamp方法。
```
Integer.numberOfLeadingZeros(n)，是返回二进制表示中，前面有多少个连续的0。这里n是2的幂，假设n = 2^x，
那么有 Integer.numberOfLeadingZeros(n) = 31 - x < 32，二进制最高的27位都是0。并且它只跟n有关，n不同，这个运算的结果也不同。  
<br/>
1 << (RESIZE_STAMP_BITS - 1)，因为RESIZE_STAMP_BITS 最小是6，因此这个运算，在不考虑变成负数的情况下，最小是 1<< 5 = 32，
二进制最低的5位都是0,进行 | 运算后，可以知道二者的二进制不重叠，可以确定，n不同时，每个rs也不同。  
<br/>
接着看下这个 else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))，这个是sizeCtl有关的。
如果按照sizeCtl本身的注释，这里应该是把sc更新为sc - 1才对，这一点就说明它本身的注释是不准确的。为了防止扩容重叠，
这里使用扩容生成戳进行了处理，尽量避免扩容重叠。  
<br/>
扩容重叠改如何理解？为什么上面的这种对sizeCtl进行的处理（下面叫做resizeStamp机制）能避免扩容重叠？  
``` java
// 来自addCount方法中触发扩容的代码
// 直接使用 -(1 + nThreads) 表示正在扩容的线程数时，去掉rs相关的代码，就变成了下面这样
while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
    int rs = resizeStamp(n);
    if (sc < 0) {
        if ((nt = nextTable) == null || transferIndex <= 0)
            break;
        if (U.compareAndSwapInt(this, SIZECTL, sc, sc - 1))
            transfer(tab, nt);
    }
    else if ...
}
```
不清楚这些代码的预期执行顺序的，可以看下字节码，截取了addCount方法的字节码的一部分
```
114 aload_0
115 invokevirtual #30 <java/util/concurrent/ConcurrentHashMap.sumCount>
118 lstore 7
120 iload_3
121 iflt 291 (+170)
124 lload 7
126 aload_0
127 getfield #25 <java/util/concurrent/ConcurrentHashMap.sizeCtl>
130 dup
131 istore 12
133 i2l
134 lcmp
135 iflt 291 (+156)
138 aload_0
139 getfield #35 <java/util/concurrent/ConcurrentHashMap.table>
142 dup
143 astore 9
145 ifnull 291 (+146)
148 aload 9
150 arraylength
151 dup
152 istore 11
154 ldc #4 <1073741824>
156 if_icmpge 291 (+135)
159 iload 11
161 invokestatic #142 <java/util/concurrent/ConcurrentHashMap.resizeStamp>
164 istore 13
166 iload 12
168 ifge 252 (+84)
171 iload 12
173 getstatic #143 <java/util/concurrent/ConcurrentHashMap.RESIZE_STAMP_SHIFT>
176 iushr
177 iload 13
179 if_icmpne 291 (+112)
182 iload 12
184 iload 13
186 iconst_1
187 iadd
188 if_icmpeq 291 (+103)
191 iload 12
193 iload 13
195 getstatic #144 <java/util/concurrent/ConcurrentHashMap.MAX_RESIZERS>
198 iadd
199 if_icmpeq 291 (+92)
202 aload_0
203 getfield #145 <java/util/concurrent/ConcurrentHashMap.nextTable>
206 dup
207 astore 10
209 ifnull 291 (+82)
212 aload_0
213 getfield #146 <java/util/concurrent/ConcurrentHashMap.transferIndex>
216 ifgt 222 (+6)
219 goto 291 (+72)
222 getstatic #13 <java/util/concurrent/ConcurrentHashMap.U>
```
根据字节码，Java中给这sc tab nt几个变量排列的执行顺序是，先执行sc = sizeCtl，判断 s>=sc，再执行tab = table，判断tab != null，
再执行nt = nextTable，判断nt == null。  
<br/>
假设有一个线程正在执行n -> 2n 的扩容，那么sizeCtl = -2，此时一个线程，叫做线程A，来尝试帮助执行扩容
（对照源码看，addCount方法的while循环，里面会先后读取sizeCtl、table、nextTable来给局部变量赋值，这中间可能会插入其他操作，
导致局部变量的值是旧值）。  
<br/>
A先执行sc= sizeCtl，此时得到sc = -2，这样判断sc < 0会为真。再执行 tab = table，此时 n -> 2n 的扩容还没完成，
tab赋值为长度为n的数组。然后线程A一段时间不执行了，等它再次执行下一句，也就是赋值操作 nt = nextTable 时，
n -> 2n的扩容早就已经完成了，并且现在又有一个的线程正在执行 2n -> 4n的扩容，此时sizeCtl仍然是-2，但是nt被赋值为长度为4n的数组。
如果还有可以申请transfer任务，最后线程A就会进入transfer方法，此时它的tab长度为n，nt长度为4n。  
<br/>
如果transfer方法中碰到了这种情况（两个数组长度不满足2倍关系），它可能会让线程A申请到的任务直接作废，就算不是直接作废，
因为线程A的tab中所有hash桶都已经被迁移完成了，这个任务实际也算是作废了。作废的任务会让最后一个退出扩容的线程来检查出来并处理掉，
这是一个预防机制。  
<br/>
但是还有很重要的一点，按照上面的假设保证不了，那就是线程A一定不能成为最后一个退出扩容的线程，有两个原因：  
1、A的tab的长度是n，实际扩容完成后，使用n重设下一次扩容阈值sizeCtl时，会让sizeCtl变成一个错误的值
（正确值的一半，这个影响不太大，下次扩容就恢复了）；  
2、A作为最后一个线程，它需要重新检查一次所有hash桶，这个是针对长度为n的tab来进行的。实际上tab中所有hash桶都被迁移完成了，
检测tab没有意义。真正需要检查是的当下的table数组，它的长度为2n，因为线程A申请到的扩容任务作废了，
当前的table数组中有一部分hash桶没有迁移，此时线程A最后检查一遍时，发现不了这个问题，也处理不了，
导致扩容完成时有部分hash桶没有被迁移，发生了扩容错误。所以必须不能让这个扩容重叠、参数错误的线程，成为最后一个退出扩容的线程。

在使用 -(1 + nThreads) 表示有nThreads个线程正在进行扩容操作时，会出现上面描述的扩容重叠，导致发生错误，很大程度上还是因为CAS操作，
因为CAS操作无法处理ABA问题。直接使用-(1 + nThreads)表示正在执行扩容的线程数时，sizeCtl会经常是一些常见的数，
不同的扩容中很容易出现相同的，这样会频繁发生ABA问题。因此不能直接像sizeCtl注释所说的，
直接使用 -(1 + nThreads) 来表示正在执行扩容的线程数目。为了避免ABA问题导致的扩容重叠，应该让sizeCtl跟table数组的长度n绑定，
并且sizeCtl能够反解出它对应的n，这样就能够保证这一轮的扩容跟另外一轮扩容，它们的执行过程中不可能出现相同的sizeCtl。
## 9. 只读遍历器Traverser
之前版本的ConcurrentHashMap，都有使用Segment代理分担各种操作，对它进行写操作会首先加锁，Segment扩容都是在写操作中触发的，
因此扩容时一定加了锁。它们的Segment的扩容也是单线程的，进行节点复制并迁移，不会更改当前的数组的任何地方
（重用的节点不用复制，但是也不会产生更改） 。也就是在扩容期间，Segment是不会有任何更改的，中间无论对这个Segment发生多少次读操作，
都不受正在进行的扩容的影响，都会看到一致的结果（因为没有全局锁，Map的遍历还是不具有一致性，期间可能会看到写操作的更改）。  
<br/>
1.8版本的扩容，采用了多线程扩容，并且很重要的一点，那就是扩容会对当前的table数组进行更改，
扩容时会在transfer完一个hash桶中的所有节点后，把一个转发节点安放进去。这会影响正在进行的读操作。
普通的读操作只读去一个hash桶中的内容，containsValue这种就会遍历读取所有hahs桶，此时就不能使用之前版本的那种什么都不考虑的遍历读，
因为此时读会跨越两个数组进行。因此写了这个内部类，它考虑了遍历读与扩容并发的情况，专门处理1.8版本中的遍历读。  
<br/>
处理的逻辑很简单，就是碰到ForwardingNode时，保存当前正在遍历的数组以及索引信息，然后在FN.nextTable上进行遍历。如果继续碰到FN，
就再保存一份信息，跳转到下下个数组进行hash桶节点的遍历。遍历完成后，返回最近一次记录的数组进行遍历，并清除掉最近一次保存的信息。
这种行为跟入栈出栈很像，因此可以使用栈结构来保存。  
<br/>
这就是TableStack的由来，它是一个简化的栈，使用链表表示的。入栈就是在链表当前头结点的前面插入一个节点，并把头结点指针指向当前节点；
出栈就是删除当前的头结点，把头结点指针指向删除的节点后面的一个节点。  
<br/>
根据transfer的性质，在FN.nextTable上进行的一次遍历只用遍历两个hash桶，index 以及 index + index。在一次出栈操作执行前，
需要遍历完这两个hash桶。 
``` java
static final class TableStack<K,V> {
    int length;
    int index;
    Node<K,V>[] tab;
    TableStack<K,V> next;
}
 
static class Traverser<K,V> {
    Node<K,V>[] tab;        // 当前数组，也就是扩容完成后的旧数组
    Node<K,V> next;         // 新数组，扩容完成后使用的数组
    TableStack<K,V> stack, spare; // 用来 保存/恢复 转发节点
    int index;              // 下一个要读取的hash桶的下标
    int baseIndex;          // 起始的下标，下界
    int baseLimit;          // 终止的下标，上界
    final int baseSize;     // 数组的长度
 
    Traverser(Node<K,V>[] tab, int size, int index, int limit) {
        this.tab = tab;
        this.baseSize = size;
        this.baseIndex = this.index = index;
        this.baseLimit = limit;
        this.next = null;
    }
 
    // 遍历器的指针往前移动到下一个有实际数据节点，并返回这个节点，如果到头就返回null
    final Node<K,V> advance() {
        Node<K,V> e;
        if ((e = next) != null) // 如果已经进入了一个非空的hash桶，直接尝试获取它的下一个节点
            e = e.next;
        for (;;) {
            Node<K,V>[] t; int i, n;  // must use locals in checks
            if (e != null) // 节点非null，直接返回
                return next = e;
            // 一些边界判断，遍历越界了表明没有了，可以直接返回null
            if (baseIndex >= baseLimit || (t = tab) == null || (n = t.length) <= (i = index) || i < 0)
                return next = null;
            if ((e = tabAt(t, i)) != null && e.hash < 0) { // 处理特殊节点
                if (e instanceof ForwardingNode) { // 转发节点，主要处理这个
                    tab = ((ForwardingNode<K,V>)e).nextTable; // 将遍历迁移到FN.nextTable新数组上进行
                    e = null;
                    pushState(t, i, n); // 入栈保存当前对tab数组的遍历信息
                    continue; // 开始新一次循环，遍历nextTable中对应的hash桶
                }
                // TreeBin时，获取红黑树所有节点的链表形式的头节点，使用链表的方式遍历，更简单
                else if (e instanceof TreeBin) 
                    e = ((TreeBin<K,V>)e).first;
                else // 保留节点，没实际数据
                    e = null;
            }
            if (stack != null) // 栈不为空
                recoverState(n); // 这里可以看做是出栈操作，得先遍历完FN.nextTable中的两个之后再出栈
            else if ((index = i + baseSize) >= n) // 栈为空，准备遍历下一个hash桶
                index = ++baseIndex; // visit upper slots if present
        }
    }
 
    // 入栈操作，保存当前对tab的遍历信息
    private void pushState(Node<K,V>[] t, int i, int n) {
        TableStack<K,V> s = spare;  // reuse if possible
        if (s != null)
            spare = s.next;
        else
            s = new TableStack<K,V>();
        s.tab = t;
        s.length = n;
        s.index = i;
        s.next = stack;
        stack = s;
    }
 
    // 可能会出栈，不出栈时，更改索引，准备遍历的是FN.nextTable中对应的第二个hash桶
    private void recoverState(int n) {
        TableStack<K,V> s; int len;
        while ((s = stack) != null && (index += (len = s.length)) >= n) {
            n = len;
            index = s.index;
            tab = s.tab;
            s.tab = null;
            TableStack<K,V> next = s.next;
            s.next = spare; // save for reuse
            stack = next;
            spare = s;
        }
        if (s == null && (index += baseSize) >= n)
            index = ++baseIndex;
    }
}
```
![](/images/posts/多线程/)  


















