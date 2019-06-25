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
## 2. ThreadLocalMap的源码实现
ThreadLocalMap提供了一种为ThreadLocal定制的高效实现，并且自带一种基于弱引用的垃圾清理机制。  
#### 2.1 存储结构
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
#### 2.2 为什么要弱引用
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

#### 2.3 类成员变量与相应方法
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
![](/images/posts/多线程/)  
ThreadLocalMap维护了Entry环形数组，数组中元素Entry的逻辑上的key为某个ThreadLocal对象（实际上是指向该ThreadLocal对象的弱引用），
value为代码中该线程往该ThreadLoacl变量实际塞入的值。
#### 2.4 构造函数
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
#### 2.5 哈希函数
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
#### 2.6 getEntry方法
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
getEntryAfterMiss方法
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










































































































