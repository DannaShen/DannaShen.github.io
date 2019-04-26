---
layout: post
title: ArrayList源码解析
categories: Java基础
description: JDK1.8之ArrayList源码分析
keywords: ArrayList源码
---

## 1.概述
- ArrayList是可以动态增长和缩减的索引序列，它是基于数组实现的List
- 该类封装了一个动态再分配的Object[]数组，每个类对象都有一个capacity属性，表示它封装的Object[]
数组的长度，当向ArrayList中添加元素时，该属性会自动增加。当想添加大量元素时，可使用
ensureCapacity一次性增加capacity，可减少重分配的次数以提高性能。
- 底层结构是数组，数组元素类型为Object类型，即可以存放所有类型的数据
- ArrayList是非线程安全
- ArrayList和Collection的关系
![](/images/posts/Java基础/集合/ArrayList和Collection关系.png) 

## 2.继承结构和层次关系
``` java
   public class ArrayList<E> extends AbstractList<E>
           implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
(1)为什么要先继承AbstractList，而又实现List接口呢？而不是让ArrayList直接实现List<E>？  
这里是有一个思想，接口中全都是抽象的方法，而抽象类中可以有抽象方法，还可以有具体的实现方法，
正是利用了这一点，让AbstractList是实现接口中一些通用的方法，而具体的类，
如ArrayList就继承这个AbstractList类，拿到一些通用的方法，然后自己在实现一些自己特有的方法，
这样一来，让代码更简洁，把继承结构最底层的类中通用的方法都抽取出来先一起实现了，减少重复代码。  
(2)为什么ArrayList还要实现?猜测应该是没有什么用。  
(3)RandomAccess接口：这个是一个标记性接口，它的作用就是用来快速随机存取，有关效率的问题，
在实现了该接口的话，那么使用普通的for循环来遍历，性能更高，例如arrayList。而没有实现该接口的话，
使用Iterator来迭代，这样性能更高，例如linkedList。
所以这个标记性只是为了让我们知道我们用什么样的方式去获取数据性能更好。  
Cloneable接口：实现了该接口，就可以使用Object.Clone()方法了。  
Serializable接口：实现该序列化接口，表明该类可以被序列化，什么是序列化？
简单的说，就是能够从类变成字节流传输，然后还能从字节流变成原来的类。  

## 3.内部属性
``` java
    // 缺省容量
    private static final int DEFAULT_CAPACITY = 10;
    // 空对象数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    // 缺省空对象数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 元素数组
    transient Object[] elementData;
    // 实际元素大小，默认为0
    private int size;
    // 最大数组容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

## 4.构造方法

``` java
　　 public ArrayList() {　　
        super();        //调用父类中的无参构造方法，父类中的是个空的构造方法
        this.elementData = EMPTY_ELEMENTDATA;//将elementData初始化为空数组
    }
```
``` java
    public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity]; //将自定义的容量大小当成初始化elementData的大小
    }
```
``` java
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();    //转换为数组
        size = elementData.length;   //数组中的数据个数
        //每个集合的toarray()的实现方法不一样，所以需要判断一下，如果不是Object[].class类型，
        //那么久需要使用ArrayList中的方法去改造一下。
        if (elementData.getClass() != Object[].class) 
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }　　
```
## 5.常用方法

#### add
添加一个特定的元素到list末尾：  
``` java
    public boolean add(E e) {    
       //确定内部容量是否足够。size是数组中数据的个数，因为要添加一个元素，所以size+1，
       ensureCapacityInternal(size + 1);  // Increments modCount!!
       //在数据中正确的位置上放上元素e，并且size++
       elementData[size++] = e;
       return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == EMPTY_ELEMENTDATA) { //判断初始化的elementData是不是空的数组
            //如果是空数组，则取mincapacity为默认的容量10和minCapacity最大值
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        //确认实际的容量，上面只是将minCapacity=10，这个方法就是真正的判断elementData是否够用
            ensureExplicitCapacity(minCapacity);
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
            modCount++;
    //minCapacity(代表最小的数据容量)如果大于了实际elementData的长度(数组new出来的大小)，
    //那么就说明elementData数组的长度不够用，不够用那么就要增加elementData的length。
    //minCapacity到底是什么呢？
    /*第一种情况：由于elementData初始化时是空的数组，那么第一次add的时候，minCapacity=size+1；
    //也就minCapacity=1，调用上面的方法将minCapacity=10，还没有改变elementData的大小。
    //第二种情况：elementData不是空的数组了，那么在add的时候，minCapacity=size+1；
    //也就是minCapacity代表着elementData中增加之后的实际数据个数，拿着它判断elementData的
    //length是否够用，如果length不够用，那么肯定要扩大容量，不然增加的这个元素就会溢出。
        if (minCapacity - elementData.length > 0)
            //arrayList能自动扩展大小的关键方法就在这里了
            grow(minCapacity);
        }
    }
    
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;//将扩充前的elementData大小给oldCapacity
        int newCapacity = oldCapacity + (oldCapacity >> 1);//newCapacity就是1.5倍oldCapacity
        //即elementData就空数组的时候，length=0，那么oldCapacity=0，newCapacity=0，
        //minCapacity=10，在这里就是真正的初始化elementData的大小了，就是为10。
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果newCapacity超过了最大的容量，就调用hugeCapacity，也就是将能给的最大值给newCapacity
        //注意这里hugeCapacity的参数是minCapacity，因为minCapacity才是必须的容量最小值，既然
        //newCapacity(扩大后的容量，不一定都能占满)已经超出了最大值，那就让minCapacity达到最大值即可
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        //新的容量大小已经确定好了，就copy数组，改变容量大小咯。
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        //如果minCapacity大于MAX_ARRAY_SIZE，则返回Integer.MAX_VALUE，反之将MAX_ARRAY_SIZE返回
        //还是超过了这个限制，就要溢出了。
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
        }
```
在特定位置添加元素：    
``` java
    public void add(int index, E element) {
        rangeCheckForAdd(index);//检查index也就是插入的位置是否合理。

        ensureCapacityInternal(size + 1); //见上面
        //这个方法就是用来在插入元素之后，要将index之后的元素都往后移一位
        //参数：源对象、源对象对象的起始位置、目标对象、目标对象的起始位置、从起始位置往后复制的长度
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        //在目标位置上存放元素
        elementData[index] = element;
        size++;//size增加1
    }
```
例子说明：    
(1)

    List<Integer> lists = new ArrayList<Integer>();
    lists.add(8);
   步骤：
   - 调用无参构造函数，this.elementData={}
   - add(8)
        -> ensureCapacityInternal(size+1)即ensureCapacityInternal(1)  
            -> ensureExplicitCapacity(1)  
                -> calculateCapacity(elementData,1)：若elementData为{}，则取10，否则返回 
                    -> 

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

