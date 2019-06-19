---
layout: post
title: 并发容器和框架之ConcurrentHashMap
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
>本篇文章以JDK1.8为前提

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
// 读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容
static final int MOVED     = -1; // hash for forwarding nodes
 
// TreeBin的hash值，TreeBin是ConcurrentHashMap中用于代理操作TreeNode的特殊节点，持有存储实际数据的红黑树的根节点
// 因为红黑树进行写入操作，整个树的结构可能会有很大的变化，这个对读线程有很大的影响，
// 所以TreeBin还要维护一个简单读写锁，这是相对HashMap，这个类新引入这种特殊节点的重要原因
static final int TREEBIN   = -2; // hash for roots of trees
 
// ReservationNode的hash值，ReservationNode是一个保留节点，就是个占位符，不会保存实际的数据，正常情况是不会出现的，
// 在jdk1.8新的函数式有关的两个方法computeIfAbsent和compute中才会出现
static final int RESERVED  = -3; // hash for transient reservations
 
// 用于和负数hash值进行 & 运算，将其转化为正数（绝对值不相等），Hashtable中定位hash桶也有使用这种方式来进行负数转正数
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


























