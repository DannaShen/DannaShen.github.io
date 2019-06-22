---
layout: post
title: 并发容器和框架之ConcurrentLinkedQueue
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. ConcurrentLinkedQueue的结构
ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，
它会添加到队列的尾部；当我们获取一个元素时，它会返回队列头部的元素。它采用了“wait-free”算法（即CAS算法）来实现，
CAS并不是一个算法，它是一个CPU直接支持的硬件指令，这也就在一定程度上决定了它的平台相关性。  
<br/>
当前常用的多线程同步机制可以分为下面三种类型：  

- volatile 变量：轻量级多线程同步机制，不会引起上下文切换和线程调度。仅提供内存可见性保证，不提供原子性。  
- CAS 原子指令：轻量级多线程同步机制，不会引起上下文切换和线程调度。它同时提供内存可见性和原子化更新保证。  
- 互斥锁：重量级多线程同步机制，可能会引起上下文切换和线程调度，它同时提供内存可见性和原子性。  

ConcurrentLinkedQueue 的非阻塞算法实现主要可概括为下面几点：  

- 使用 CAS 原子指令来处理对数据的并发访问，这是非阻塞算法得以实现的基础。  
- head/tail 并非总是指向队列的头 / 尾节点，也就是说允许队列处于不一致状态。这个特性把入队 / 出队时，原本需要一起原子化执行的
两个步骤分离开来，从而缩小了入队 / 出队时需要原子化更新值的范围到唯一变量。这是非阻塞算法得以实现的关键。  
- 以批处理方式来更新head/tail，从整体上减少入队 / 出队操作的开销。  

![](/images/posts/多线程/)  

ConcurrentLinkedQueue由head节点和tail节点组成，每个节点（Node）由节点元素（item）和指向下一个节点的引用(next)组成，
节点与节点之间就是通过这个next关联起来，从而组成一张链表结构的队列。默认情况下head节点存储的元素为空，tail节点等于head节点。  
## 2. 源码分析
#### 2.1 前提
1. 基本原则不变性  

- 整个队列中一定会存在一个 node(该node.next = null), 并且仅存在一个, 但tail引用不一定指向它  
- 队列中所有 item != null 的节点, head一定能够到达; cas 设置 node.item = null, 意味着这个节点被删除  

2. head引用  
不变性  

- 所有的有效节点通过 succ() 方法都可达  
- head != null  
- (tmp = head).next != tmp || tmp != head (其实就是 head.next != head)  
 
<br/>

可变性  

- head.item 可能是 null, 也可能不是 null  
- 允许 tail 滞后于 head, 也就是调用 succ() 方法, 从 head 不可达tail  

3. tail 引用  
不变性  

- tail 节点通过succ()方法一定到达队列中的最后一个节点(node.next = null)
- tail != null
 
可变性  

- tail.item 可能是 null, 也可能不是 null  
- 允许 tail 滞后于 head, 也就是调用 succ() 方法, 从 head 不可达tail  
- tail.next 可能指向 tail  

4. ConcurrentLinkedQueue异于一般queue的特点:  

- head 与 tail 都有可能指向一个 (item = null) 的节点  
- 如果 queue 是空的, 则所有 node.item = null  
- queue刚刚创建时 head = tail = dummyNode  
- head/tail 的 item/next 的操作都是通过 CAS  

#### 2.2 内部节点 Node
``` java
public class Node<E> {
    volatile E item;
    volatile Node<E> next;

    Node(E item){
        
        //将 Node 对象的指定 itemOffset 偏移量设置 一个引用值
        unsafe.putObject(this, itemOffset, item);
    }

    boolean casItem(E cmp, E val){
        
        //原子性的更新 item 值
        return unsafe.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    void lazySetNext(Node<E> val){
        
        //调用这个方法和putObject差不多, 只是这个方法设置后对应的值的可见性不一定得到保证,
        //这个方法能起这个作用, 通常是作用在 volatile field上, 也就是,下面中的参数 val 是被volatile修饰
        unsafe.putOrderedObject(this, nextOffset, val);
    }

    //原子性的更新 nextOffset 上的值
    boolean casNext(Node<E> cmp, Node<E> val){
        return unsafe.compareAndSwapObject(this, nextOffset, cmp, val);
    }

    private static Unsafe unsafe;
    private static long itemOffset;
    private static long nextOffset;

    static {
        try {
            unsafe = UnSafeClass.getInstance();
            Class<?> k = Node.class;
            itemOffset = unsafe.objectFieldOffset(k.getDeclaredField("item"));
            nextOffset = unsafe.objectFieldOffset(k.getDeclaredField("next"));
        }catch (Exception e){

        }
    }
}
```
#### 2.3  内部属性、构造方法及其他辅助方法
##### 2.3.1 内部属性和构造方法
``` java
//head 节点
private transient volatile Node<E> head;
//tail 
private transient volatile Node<E> tail;

public ConcurrentLinkedList() {
    //默认会构造一个 dummy 节点，dummy 的存在是防止一些特殊复杂代码的出现 
    head = tail = new Node<E>(null);
}
```
#### 2.3.2 查询后继节点 succ()
``` java
//获取 p 的后继节点, 若 p.next = p (updateHead 操作导致的), 
//则说明 p 已经 fall off queue, 需要 jump 到 head
final Node<E> succ(Node<E> p){
    Node<E> next = p.next;
    return (p == next)? head : next;
}
``` 
获取一个节点的后继节点不仅是 node.next , 还有特殊情况, 就是tail 指向一个哨兵节点 (node.next = node); 
代码的注释中提到了 哨兵节点是 updateHead 导致的, 那我们来看 updateHead方法.  
#### 2.3.3 特别的更新头节点方法 updateHead()
``` java

//将节点 p设置为新的节点(这是原子操作),
//之后将原节点的next指向自己, 直接变成一个哨兵节点(为queue节点删除及garbage做准备)
final void updateHead(Node<E> h, Node<E> p){
    if(h != p && casHead(h, p)){
        h.lazySetNext(h);
    }
}
```
主要这个 h.lazySetNext(h), 将 h.next -> h 直接变成一个哨兵节点, 这种lazySetNext主要用于无阻塞数据结构的nulling out。
#### 2.4 入队操作 offer()
``` java
//在队列的末尾插入指定的元素
public boolean offer(E e){
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e); // 1. 构建一个 node

    for(Node<E> t = tail, p = t;;){         // 2. 初始化变量 p = t = tail
        Node<E> q = p.next;                 // 3. 获取 p 的next
        if(q == null){                      // q == null, 说明 p 是 last Node
            if(p.casNext(null, newNode)){   // 4. 对 p 进行 cas 操作, newNode -> p.next
                // 5. 每经过一次p =q操作(向后遍历节点，即p的next赋给p), 
                // 则 p != t 成立, 这个也说明tail滞后于head的体现
                // 当之前的尾节点和现在插入的节点之间有一个节点时才设置尾节点，并不是每一次都cas设置
                if(p != t){ 
                    casTail(t, newNode); 
                }
                return true;
            }
        }
        // 6. (p == q) 成立,则说明p是pool()时调用 "updateHead" 导致的(删除头节点),
        // 将head的next设置为新的head,所以这里需要重新找新的head
        else if(p == q){  
            /** 1. 大前提 p 是已经被删除的节点
             *  2. 判断 tail 是否已经改变
             *      1) tail 已经变化, 则说明 tail 已经重新定位
             *      2) tail 未变化, 而 tail 指向的节点是要删除的节点, 所以让 p 指向 head
             *  判断尾节点是否有变化
             *  1. 尾节点变化, 则用新的尾节点
             *  2. 尾节点没变化, 将 tail 指向head
             */
            p = (t != (t = tail))? t : head;
        }else{
            // 7. 寻找尾节点
            p = (p != t && (t != (t = tail))) ? t : q;
        }
    }
}
```
#### 2.5 出队列操作 poll()
``` java
public E poll(){
    restartFromHead:
    for(;;){        
        for(Node<E> h = head, p = h, q;;){ // 获取头结点
            E item = p.item;
            if(item != null && p.casItem(item, null)){ // item不为null ，使用cas设置item为空
                
                if(p != h){ // 3. 若此时的 p != h, 则更新 head(什么时候p != h? -> 执行第8步后)
                    // 4. 进行 cas 更新 head ; "(q = p.next) != null" 怕出现p此时是尾节点了; 
                    // 在 ConcurrentLinkedQueue 中正真的尾节点只有1个(必须满足node.next = null)
                    //不是每次都更新，头结点item为null时，下个节点就必须更新，不为null时，规则和更新尾节点一样
                    updateHead(h, ((q = p.next) != null)? q : p); 
                }
                return item;
            }
            else if((q = p.next) == null){  // 5. 如果p节点的下一节点为null, 则表明队列为空
                 // 6. 这一步除了更新head 外, 还是helpDelete删除队列操作 删除 p 之前的节点
,                updateHead(h, p); 
                return null;
            }
            // 7. p == q -> 说明别的线程调用了updateHead，自己的next 指向了自己，重新循环，获取最新的头结点
            else if(p == q){ 
                continue restartFromHead;
            }else
                // 如果下一个元素不为空，则将头节点的下一个节点设置成头节点
                p = q; 
        }
    }
}
```


















