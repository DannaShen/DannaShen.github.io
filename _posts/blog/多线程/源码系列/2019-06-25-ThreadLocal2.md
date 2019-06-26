---
layout: post
title: ThreadLocal
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. ThreadLocal的理解
#### 1.1 什么是ThreadLocal
ThreadLocal类可以理解为线程本地变量。也就是说如果定义了一个ThreadLocal，每个线程往这个ThreadLocal中读写是线程隔离，
互相之间不会影响的。它提供了一种将可变数据通过每个线程有自己的独立副本从而实现线程封闭的机制。  
#### 1.2 实现思路
Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，也就是说每个线程有一个自己的ThreadLocalMap。
ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值
（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。每个线程在往某个ThreadLocal里塞值的时候，
都会往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。  
#### 1.3 ThreadLocal实例介绍
定义两个ThreadLocal实例：  
``` java
static ThreadLocal<User> threadLocal_1 = new ThreadLocal<>();
static ThreadLocal<Client> threadLocal_2 = new ThreadLocal<>();
```
我们分别在三个线程中使用 ThreadLocal，伪代码如下：  
``` java
// thread-1中
threadLocal_1.set(user_1);
threadLocal_2.set(client_1);

// thread-2中
threadLocal_1.set(user_2);
threadLocal_2.set(client_2);

// thread-3中
threadLocal_2 .set(client_3);
```
这三个线程都在运行中，那此时各线程中的存数数据应该如下图所示：  
![](/images/posts/多线程/源码系列/ThreadLocal-例图.png)  
每个线程持有自己的 ThreadLocalMap，ThreadLocalMap 初始容量为16（即图中的16个槽位），在调用ThreadLocal 的 set 方法时，
将以 ThreadLocal 为 Key 存储在 本线程的 ThreadLocalMap 里面，ThreadLocalMap 的 Value 为Object 类型，
实际类型由 ThreadLocal 定义。
## 2. ThreadLocal的源码实现
#### 2.1 set方法
``` java
public void set(T value) {
    Thread t = Thread.currentThread(); // 获取当前线程
    ThreadLocalMap map = getMap(t); // 拿到当前线程的 ThreadLocalMap
    if (map != null) // 判断 ThreadLocalMap 是否存在
        map.set(this, value); // 调用 ThreadLocalMap 的 set 方法
    else
        createMap(t, value); // 创建 ThreadLocalMap
}
```
#### 2.2 get方法
``` java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); //调用  ThreadLocalMap 的 getEntry 方法
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue(); // 如果还没有设置，可以用子类实现 initialValue ，自定义初始值。
}
```
#### 2.3 remove方法
``` java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this); // 调用 ThreadLocalMap 的 remove方法
}
```
这里ThreadLocal 的几个public方法，其实所有工作最终都落到了 ThreadLocalMap 的头上，
ThreadLocal 仅仅是从当前线程取到 ThreadLocalMap 而已，具体由ThreadLocalMap执行。  
## 3. ThreadLocalMap的源码实现
ThreadLocalMap提供了一种为ThreadLocal定制的高效实现，并且自带一种基于弱引用的垃圾清理机制。  
#### 3.1 存储结构
既然是个map（注意不要与java.util.map混为一谈，这里指的是概念上的map），当然得要有自己的key和value，
我们可以将其简单视作key为ThreadLocal，value为实际放入的值。之所以说是简单视作，
因为实际上ThreadLocal中存放的是ThreadLocal的弱引用。我们来看看ThreadLocalMap里的节点是如何定义的。  
``` java
static class Entry extends WeakReference<java.lang.ThreadLocal<?>> {
    // 往ThreadLocal里实际塞入的值
    Object value;

    Entry(java.lang.ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
Entry便是ThreadLocalMap里定义的节点，它继承了WeakReference类，定义了一个类型为Object的value，用于存放塞到ThreadLocal里的值。  
#### 3.2 为什么要弱引用
因为如果这里使用普通的key-value形式来定义存储结构，实质上就会造成节点的生命周期与线程强绑定，只要线程没有销毁，
那么节点在GC分析中一直处于可达状态，没办法被回收，而程序本身也无法判断是否可以清理节点。弱引用是Java中四档引用的第三档，
比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次GC。当某个ThreadLocal已经没有强引用可达，
则随着它被垃圾回收，在ThreadLocalMap里对应的Entry的键值会失效，这为ThreadLocalMap本身的垃圾清理提供了便利。  
<br/>
补充强引用、软引用、弱引用和虚引用之间的关系：  

- 强引用：在程序代码之中普遍存在的，类似“Object obj=new Object()”这类的引用，只要强引用还存在，
垃圾收集器永远不会回收掉被引用的对象。  
- 软引用：用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，
将会把这些对象列进回收范围之中 进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。
在JDK 1.2之后，提供了SoftReference类来实现软引用。  
- 弱引用：用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。
当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK 1.2之后，提供了WeakReference类来实现弱引用。  
- 虚引用：也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，
也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
在 JDK 1.2之后，提供了PhantomReference类来实现虚引用。  
#### 3.3 为什么采用开放地址法
**开放地址法**：容易产生堆积问题；不适于大规模的数据存储；散列函数的设计对冲突会有很大的影响；
插入时可能会出现多次冲突的现象，删除的元素是多个冲突元素中的一个，需要对后面的元素作处理，实现较复杂；
结点规模很大时会浪费很多空间；  
<br/>
**链地址法**：处理冲突简单，且无堆积现象，平均查找长度短；链表中的结点是动态申请的，适合构造表不能确定长度的情况；
相对而言，拉链法的指针域可以忽略不计，因此较开放地址法更加节省空间。插入结点应该在链首，删除结点比较方便，
只需调整指针而不需要对其他冲突元素作调整。  
<br/>
ThreadLocalMap 为什么采用开放地址法？  
个人认为由于 ThreadLocalMap 的 hashCode 的精妙设计，使hash冲突很少，并且 Entry 继承 WeakReference， 很容易被回收，
并开方地址可以节省一些指针空间；然而恰恰由于开方地址法的使用，使在处理hash冲突时的代码很难懂，
比如在replaceStaleEntry,cleanSomeSlots，expungeStaleEntry 等地方，然而真正调用这些方法的几率却比较小；

#### 3.3 类成员变量与相应方法
``` java

//初始容量，必须为2的幂
private static final int INITIAL_CAPACITY = 16;

//Entry表，大小必须为2的幂
private Entry[] table;

//表里entry的个数
private int size = 0;

//重新分配表大小的阈值，默认为0
private int threshold; 
```
可以看到，ThreadLocalMap维护了一个Entry表或者说Entry数组，并且要求表的大小必须为2的幂，
同时记录表里面entry的个数以及下一次需要扩容的阈值。  
为什么必须是2的幂？下面再回答。
<br/>

``` java

//设置resize阈值以维持最坏2/3的装载因子
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

//环形意义的下一个索引
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

//环形意义的上一个索引
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```
ThreadLocal需要维持一个最坏2/3的负载因子。  
<br/>
ThreadLocal有两个方法用于得到上一个/下一个索引，注意这里实际上是环形意义下的上一个与下一个。  

由于ThreadLocalMap使用线性探测法来解决散列冲突，所以实际上Entry[]数组在程序逻辑上是作为一个环形存在的。

至此，我们已经可以大致勾勒出ThreadLocalMap的内部存储结构。虚线表示弱引用，实线表示强引用。  
![](/images/posts/多线程/源码系列/TheadLocal-ThreadLocalMap图.png)  
ThreadLocalMap维护了Entry环形数组，数组中元素Entry的逻辑上的key为某个ThreadLocal对象（实际上是指向该ThreadLocal对象的弱引用），
value为代码中该线程往该ThreadLoacl变量实际塞入的值。
#### 3.4 构造函数
``` java
/**
 * 构造一个包含firstKey和firstValue的map。
 * ThreadLocalMap是惰性构造的，所以只有当至少要往里面放一个元素的时候才会构建它。
 */
ThreadLocalMap(java.lang.ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化table数组
    table = new Entry[INITIAL_CAPACITY];
    // 用firstKey的threadLocalHashCode与初始大小16取模得到哈希值
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 初始化该节点
    table[i] = new Entry(firstKey, firstValue);
    // 设置节点表大小为1
    size = 1;
    // 设定扩容阈值
    setThreshold(INITIAL_CAPACITY);
}
```
#### 3.5 哈希函数
上面构造函数中的 ```int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);``` 这一行代码。  
<br/>
ThreadLocal类中有一个被final修饰的类型为int的threadLocalHashCode，它在该ThreadLocal被构造的时候就会生成，
相当于一个ThreadLocal的ID
``` java
private final int threadLocalHashCode = nextHashCode();

//生成hash code间隙为这个魔数，可以让生成出来的值或者说ThreadLocal的ID较为均匀地分布在2的幂大小的数组中。
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
可以看出，它是在上一个被构造出的ThreadLocal的ID(threadLocalHashCode)的基础上加上一个魔数0x61c88647的。  
<br/>
这个魔数的选取与斐波那契散列有关，0x61c88647对应的十进制为1640531527。斐波那契散列的乘数可以用
(long) ((1L << 31) * (Math.sqrt(5) - 1))得到2654435769，如果把这个值给转为带符号的int，则会得到-1640531527。换句话说
(1L << 32) - (long) ((1L << 31) * (Math.sqrt(5) - 1))得到的结果就是1640531527也就是0x61c88647。通过理论与实践，
当我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。  
<br/>
ThreadLocalMap使用的是线性探测法，均匀分布的好处在于很快就能探测到下一个临近的可用slot，从而保证效率。
这就回答了上文抛出的为什么大小要为2的幂的问题。为了优化效率。  
<br/>
对于& (INITIAL_CAPACITY - 1)，因为对于2的幂作为模数取模，可以用&(2^n-1)来替代%2^n，位运算比取模效率高很多。至于为什么，
因为对2^n取模，只要不是低n位对结果的贡献显然都是0，会影响结果的只能是低n位。  
<br/>
可以说在ThreadLocalMap中，形如key.threadLocalHashCode & (table.length - 1)（其中key为一个ThreadLocal实例）
这样的代码片段实质上就是在求一个ThreadLocal实例的哈希值，只是在源码实现中没有将其抽为一个公用函数。
#### 3.6 getEntry方法
这个方法会被ThreadLocal的get方法直接调用，用于获取map中某个ThreadLocal存放的值。  
``` java
private Entry getEntry(ThreadLocal<?> key) {
    // 根据key这个ThreadLocal的ID来获取索引，也即哈希值
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 对应的entry存在且未失效且弱引用指向的ThreadLocal就是key，则命中返回
    if (e != null && e.get() == key) {
        return e;
    } else {
        // 因为用的是线性探测，所以往后找还是有可能能够找到目标Entry的。
        return getEntryAfterMiss(key, i, e);
    }
}

```
##### 3.6.1 getEntryAfterMiss方法
``` java
调用getEntry未直接命中的时候调用此方法
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
   
    
    // 基于线性探测法不断向后探测直到遇到空entry。
    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 找到目标
        if (k == key) {
            return e;
        }
        if (k == null) {
            // 该entry对应的ThreadLocal已经被回收，调用expungeStaleEntry来清理无效的entry
            expungeStaleEntry(i);
        } else {
            // 环形意义下往后面走
            i = nextIndex(i, len);
        }
        e = tab[i];
    }
    return null;
}
```
##### 3.6.2 expungeStaleEntry方法
``` java
/**
 * 清除连续段。
 * 这个函数是ThreadLocal中核心清理函数，它做的事情很简单：
 * 就是从staleSlot开始遍历，将无效（弱引用指向的对象被回收）清理，即对应entry中的value置为null，
 * 将指向这个entry的table[i]置为null，直到扫到空entry就停止。
 * 另外，在过程中还会对非空的entry作rehash。
 * 可以说这个函数的作用就是从staleSlot开始清理连续段中的slot
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 因为entry对应的ThreadLocal已经被回收，value设为null，显式断开强引用
    tab[staleSlot].value = null;
    // 显式设置该entry为null，以便垃圾回收
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 清理对应ThreadLocal已经被回收的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            /*
             * 对于还没有被回收的，需要做一次rehash。
             * 如果得到的索引h不为当前位置i，则从h向后线性探测到第一个空的slot，把当前的entry给挪过去。
             */
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {//即e=tab[i],k=e.get(),h=k取hashcode得到的索引，按说h==i，但是现在h!=i，
                         //说明i是当时h经过线性探测得到的，此时让i位置的entry为null，然后从h位置往后找
                         //直到遇到为null的位置，把e放到那个位置即可
                tab[i] = null;
                
                while (tab[h] != null) {
                    h = nextIndex(h, len);
                }
                tab[h] = e;
            }
        }
    }
    // 返回staleSlot之后第一个空的slot索引
    return i;
}
```
从ThreadLocal读一个值可能遇到的情况：  
根据入参threadLocal的threadLocalHashCode对表容量取模得到index  

- 如果index对应的slot就是要读的threadLocal，则直接返回结果  
- 调用getEntryAfterMiss线性探测，过程中每碰到无效slot，调用expungeStaleEntry进行段清理；如果找到了key，则返回结果entry  
- 没有找到key，返回null  

#### 3.7 set方法
``` java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    // 线性探测
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 找到对应的entry
        if (k == key) {// key 相同，则覆盖value
            e.value = value;
            return;
        }
        // 替换失效的entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);// 新增 Entry
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) {// 清除一些过期的值，并判断是否需要扩容
        rehash();
    }
}
```
##### 3.7.1 replaceStaleEntry方法
``` java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 向前扫描，查找最前的一个无效slot
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len)) {
        if (e.get() == null) {
            slotToExpunge = i;
        }
    }

    // 向后遍历table，找到key找到 key 或者 直到 遇到null 的slot 才终止循环。找到key时与前面无效的slot交换
    for (int i = nextIndex(staleSlot, len);(e = tab[i]) != null;i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 找到了key，将其与无效的slot交换
        if (k == key) {
            // 更新对应slot的value值
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            /*
             * 如果找到了key才走到这。
             * 如果在前面扫描无效slot中，若存在无效的slot，则以那个位置作为清理的起点，
             * 否则则以当前的i作为清理起点
             */
            if (slotToExpunge == staleSlot) {
                slotToExpunge = i;
            }
            // 从slotToExpunge开始做一次连续段的清理，再做一次启发式清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果当前的slot已经无效，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot) {
            slotToExpunge = i;
        }
    }

    // 如果key在table中不存在，则在原地放一个即可
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 在探测过程中如果发现任何无效slot，则做一次清理（连续段清理+启发式清理）
    if (slotToExpunge != staleSlot) {
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
}
```
##### 3.7.2 cleanSomeSlots方法
``` java
/**
 * 启发式地清理slot。它执行 对数 数量的扫描，是一种基于不扫描（快速但保留垃圾）和所有元素扫描之间的平衡。
 * i对应entry是非无效；n是用于控制控制扫描次数的
 * 正常情况下如果log n次扫描没有发现无效slot，函数就结束了
 * 但是如果发现了无效的slot，将n置为table的长度len，做一次连续段的清理
 * 再从下一个空的slot开始继续扫描
 * 
 * 这个函数有两处地方会被调用，一处是插入的时候可能会被调用，另外个是在替换无效slot的时候可能会被调用，
 * 区别是前者传入的n为元素个数，后者为table的容量
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // i在任何情况下自己都不会是一个无效slot，所以从下一个开始判断
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            // 扩大扫描控制因子
            n = len;
            removed = true;
            // 清理一个连续段
            i = expungeStaleEntry(i);
        }
    } while ((n >>>= 1) != 0);
    return removed;
}
```
##### 3.7.3 rehash方法
``` java
private void rehash() {
    // 做一次全量清理
    expungeStaleEntries();

    /*
     * 因为做了一次清理，所以size很可能会变小。
     * ThreadLocalMap这里的实现是调低阈值来判断是否需要扩容，
     * threshold默认为len*2/3，所以这里的threshold - threshold / 4相当于len/2，这样提前触发resize
     */
    if (size >= threshold - threshold / 4) {
        resize();
    }
}
```
##### 3.7.4 expungeStaleEntries方法
``` java
//全量清理
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null) {
            //expungeStaleEntry执行过程中是把连续段内所有无效slot都清理了一遍了。
            expungeStaleEntry(j);
        }
    }
}
```
##### 3.7.5 resize方法
``` java
//扩容，因为需要保证table的容量len为2的幂，所以扩容即扩大2倍
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; 
            } else {
                // 线性探测来存放Entry
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null) {
                    h = nextIndex(h, newLen);
                }
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
ThreadLocal的set方法可能会有的情况  

- 探测过程中slot都不无效，并且顺利找到key所在的slot，直接替换即可  
- 探测过程中发现有无效slot，调用replaceStaleEntry，效果是最终一定会把key和value放在这个slot，并且会尽可能清理无效slot  
    - 在replaceStaleEntry过程中，如果找到了key，则做一个swap把它放到那个无效slot中，value置为新值  
    - 在replaceStaleEntry过程中，没有找到key，直接在无效slot原地放entry  
- 探测没有发现key，则在连续段末尾的后一个空位置放上entry，这也是线性探测法的一部分。放完后，做一次启发式清理，
如果没清理出去key，并且当前table大小已经超过阈值了，则做一次rehash，
rehash函数会调用一次全量清理slot方法也即expungeStaleEntries，如果完了之后table大小超过了threshold - threshold / 4，
则进行扩容2倍
#### 3.8 remove方法
``` java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            // 显式断开弱引用
            e.clear();
            // 进行段清理
            expungeStaleEntry(i);
            return;
        }
    }
}
```
#### 3.9 ThreadLocalMap 的 value 清理触发时间总结
1. set(ThreadLocal<?> key, Object value)  
若无hash冲突，则先向后检测log2(N)个位置，发现过期 slot 则清除，如果没有任何 slot 被清除，则判断 size >= threshold，
超过阀值会进行 rehash()，rehash()会清除所有过期的value；  
2. getEntry(ThreadLocal<?> key)  (ThreadLocal 的 get() 方法调用)  
如果没有直接在hash计算的 slot 中找到entry， 则需要向后继续查找(直到null为止)，查找期间发现的过期 slot 会被清除；  
3. remove(ThreadLocal<?> key)  
remove 不仅会清除需要清除的 key，还是清除hash冲突的位置的已过期的 key；  


































































































