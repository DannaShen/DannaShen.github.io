---
layout: post
title: LinkedList源码解析
categories: Java基础
description: JDK1.8之LinkedList源码分析
keywords: LinkedList源码
---

## 1.概述
LinkedList是线程不安全的，允许元素为null的双向链表。其底层数据结构是链表：
``` java
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, 
                                                            Cloneable, java.io.Serializable
```
它实现了Deque<E>,所以它也可以作为一个双端队列。和ArrayList比，没有实现RandomAccess所以其以下标，
随机访问元素速度较慢。因其底层数据结构是链表，所以它的增删只需要移动指针即可，故时间效率较高。
不需要批量扩容，也不需要预留空间，所以空间效率比ArrayList高。  
缺点是随机访问元素时，时间效率很低，虽然底层在根据下标查询Node的时，会根据index判断目标Node在前半
段还是后半段，然后决定是顺序还是逆序查询，以提升时间效率。不过随着n的增大，总体时间效率依然很低。

## 2.属性
``` java
    transient int size = 0;//集合元素数量
    transient Node<E> first;//链表头节点
    transient Node<E> last;//链表尾节点
    public LinkedList() {
    }
    //调用addAll(Collection<? extends E> c)  将集合c所有元素插入链表中
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

## 3.内部节点Node结构
``` java
    private static class Node<E> {
        E item;//元素值
        Node<E> next;//后置节点
        Node<E> prev;//前置节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
## 4.常用方法

#### addAll
尾部批量增加：  
``` java
    public boolean addAll(Collection<? extends E> c) {
            return addAll(size, c);//以size为插入下标，插入集合c中所有元素
    }
```
在index处插入集合c中所有元素：    
``` java
    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);//检查越界 [0,size] 闭区间

        Object[] a = c.toArray();//拿到目标集合数组
        int numNew = a.length;//新增元素的数量
        if (numNew == 0)
            return false;

        Node<E> pred, succ;//index节点的前置节点，后置节点
        if (index == size) {//在链表尾部追加数据
            succ = null;//size节点（队尾）的后置节点一定是null
            pred = last;//前置节点是队尾(就是在队尾后添加)
        } else {
            succ = node(index);//取出index节点，作为后置节点
            pred = succ.prev;//前置节点是，index节点的前一个节点
        }
        //链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。
        // 对比ArrayList是通过System.arraycopy完成批量增加的
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)//如果前置节点是空，让新节点为头结点
                first = newNode;
            else
                pred.next = newNode;//前置节点的后置为新节点
            pred = newNode;//此时前置节点为插入的新节点
        }

        if (succ == null) {//如果后置节点为空，让此时的前置节点为尾节点
            last = pred;
        } else {
            pred.next = succ;//让此时的前置节点的后置为此时的后置节点
            succ.prev = pred;//此时的后置节点的前置为此时的前置节点
        }

        size += numNew;
        modCount++;
        return true;
    }
```
查找index处的node结点  
``` java
    //根据index处于前半段还是后半段 进行一个折半，以提升查询效率
    //如size=8，index=3，当i=index-1时，x取next，便是index处的值
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

总结：  
- 链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。对比ArrayList是通过
System.arraycopy完成批量增加的。
- 通过下标获取某个node 的时候，会根据index处于前半段还是后半段进行一个折半，以提升查询效率。

#### 插入单个节点 add
在尾部插入一个节点  
``` java
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    // 生成新节点并插入到链表尾部，更新last/first节点   
    void linkLast(E e) {
        final Node<E> l = last;//记录原尾部节点
        final Node<E> newNode = new Node<>(l, e, null);//以原尾部节点为前置节点生成新节点
        last = newNode;//新节点设置为尾部节点
        if (l == null)//原尾部节点为空，则链表为空，则设置新节点为首节点
            first = newNode;
        else//否则原尾部节点的后置为新节点
            l.next = newNode;
        size++;//数量加1
        modCount++;
    }
```
在index处插入一个节点
``` java
   public void add(int index, E element) {
       checkPositionIndex(index);//检查下标是否越界[0,size]

       if (index == size)//在尾节点后插入
           linkLast(element);
       else              //在中间插入
           linkBefore(element, node(index));//在index处之前插入
   } 
   //在succ节点前，插入一个新节点e
   void linkBefore(E e, Node<E> succ) {
       // assert succ != null;
       final Node<E> pred = succ.prev;//获取succ的前置节点
       //以前置和后置及当前e节点创建一个新节点
       final Node<E> newNode = new Node<>(pred, e, succ);
       //后置节点的前置设置为新节点
       succ.prev = newNode;
       if (pred == null)//前置节点为空，则为空表，将新结点设置为首节点
           first = newNode;
       else//否则前置节点的后置设置为新节点
           pred.next = newNode;
       size++;
       modCount++;
   }
```
#### 删除 remove
删除index处节点
``` java
    public E remove(int index) {
        checkElementIndex(index);//检查是否越界 下标[0,size)
        return unlink(node(index));//从链表上删除某节点
    }
```
从链表上删除x节点
``` java
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;//当前节点的元素值
        final Node<E> next = x.next;//当前节点的后置节点
        final Node<E> prev = x.prev;//当前节点的前置节点

        if (prev == null) {//前置节点为空，即链表为空，则当前节点的后置节点为首节点
            first = next;
        } else {//否则前置节点的后置指向后置节点
            prev.next = next;
            x.prev = null;//x的前置 置为空
        }

        if (next == null) {//后置节点为空，即当前节点为尾节点，把前置节点置为尾节点
            last = prev;
        } else {//否则后置节点的前置指向前置节点
            next.prev = prev;
            x.next = null;//x的后置 置为空
        }

        x.item = null;//x的元素值置空
        size--;
        modCount++;
        return element;
    }
```
删除链表中指定节点：
``` java
    //因为要考虑 null元素，也是分情况遍历
    public boolean remove(Object o) {
        if (o == null) {//如果要删除的是null节点(从remove和add 里 可以看出，允许元素为null)
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```

#### toArray
``` java
    public Object[] toArray() {
        //new 一个新数组 然后遍历链表，将每个元素存在数组里，返回
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }
```
## 4.总结

LinkedList 是双向列表。  
- 链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。对比ArrayList是通过
System.arraycopy完成批量增加的。增加一定会修改modCount。  
- 通过下标获取某个node 的时候，会根据index处于前半段还是后半段 进行一个折半，以提升查询效率  
- 删也一定会修改modCount。 按下标删，也是先根据index找到Node，然后去链表上unlink掉这个Node。 
按元素删，会先去遍历链表寻找是否有该Node，如果有，去链表上unlink掉这个Node。  
- 改也是先根据index找到Node，然后替换值。改不修改modCount。  
- 查本身就是根据index找到Node。  
- 所以它的CRUD操作里，都涉及到根据index去找到Node的操作。

