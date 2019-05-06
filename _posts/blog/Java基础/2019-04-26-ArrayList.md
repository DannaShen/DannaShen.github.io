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
    //第一种情况：由于elementData初始化时是空的数组，那么第一次add的时候，minCapacity=size+1；
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
``` java
    List<Integer> lists = new ArrayList<Integer>();
    lists.add(8);
```    
    
步骤：
- 调用无参构造函数ArrayList()，elementData={}。
- add(8)里面调用ensureCapacityInternal(size+1)，size为elementData实际的元素个数，
&ensp;即ensureCapacityInternal(1)，然后在elementData[size++] = e，这里e就是8。
- ensureCapacityInternal方法里面调用  
            * ensureExplicitCapacity(calculateCapacity(elementData, minCapacity))
- calculateCapacity(elementData,minCapacity)计算最小容量，里面判断elementData是否为{},如果是，
&ensp;返回minCapacity和DEFAULT_CAPACITY的最大值，否则返回minCapacity，这里返回DEFAULT_CAPACITY,即8
- ensureExplicitCapacity判断calculateCapacity计算出来的最小容量(1)是否大于elementData数组的长度(0)，
&ensp;如果大于，调用grow(minCapacity)扩展最小容量,否则什么也不做，
&ensp;这里调用grow(minCapacity)，这里的minCapacity为1。
- grow(minCapacity),旧容量为elementData的长度，将新容量赋值为旧容量的1.5倍，判断新容量是否小于最小容量，
&ensp;如果小于，将最小容量赋值给新容量；接着判断新容量是否是否大于容量的最大值，如果大于，则调用
&ensp;hugeCapacity(minCapacity)，之后调用Arrays.copyOf(elementData, newCapacity)复制，
第二个自变量指定要建立的新数组长度，如果新数组的长度超过原数组的长度，则保留数组默认值。
- hugeCapacity方法就是把Int的最大值给minCapacity。因为新容量已经超过最大值了，所以这时就把所需要的最小容量
&ensp;返回，赋给新容量，当然，如果此时新容量还是不够，只能溢出了。
<br/>
<br/>
<br/>
<br/>
(2)

``` java
    List<Integer> lists = new ArrayList<Integer>(6);
    lists.add(8);
```

步骤：
- 调用ArrayList(int initialCapacity)构造函数。 
- 若initialCapacity为0，则elementData={},否则elementData=new Object[initialCapacity]，
&ensp;这里initialCapacity为6。
- add(8)里面调用ensureCapacityInternal(size+1)，size为elementData实际的元素个数，
&ensp;即ensureCapacityInternal(1)，然后在elementData[size++] = e，这里e就是8。
- ensureCapacityInternal(minCapacity)里面调用  
              * ensureExplicitCapacity(calculateCapacity(elementData, minCapacity))
- calculateCapacity(elementData,minCapacity)计算最小容量，里面判断elementData是否为{},如果是，
&ensp;返回minCapacity和DEFAULT_CAPACITY的最大值，否则返回minCapacity，这里返回1
- ensureExplicitCapacity判断calculateCapacity计算出来的最小容量(1)是否大于elementData数组的长度(6)，
&ensp;如果大于，调用grow(minCapacity(7))扩展最小容量,否则什么也不做，这里什么都不做。


#### remove

在指定位置删除元素：  
``` java
public E remove(int index) {
    rangeCheck(index);//检查index的合理性

    modCount++;//这个作用很多，比如用来检测快速失败的一种标志。
    E oldValue = elementData(index);//通过索引直接找到该元素

    int numMoved = size - index - 1;//计算要移动的位数。
    if (numMoved > 0)
    System.arraycopy(elementData, index+1, elementData, index,numMoved);
    //将--size上的位置赋值为null，让gc(垃圾回收机制)更快的回收它。
    elementData[--size] = null; // clear to let GC do its work
    //返回删除的元素。
    return oldValue;
    }
```

删除某个元素：  
``` java
     public boolean remove(Object o) {
        if (o == null) {//这里可以看出，ArrayList可以存放null值
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```
批量删除/判断两个集合是否有交集
``` java
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;//将原集合，记名为A
    int r = 0, w = 0;//r用来控制循环，w：标记两个集合公共元素的个数
    //设置标志位
    boolean modified = false;
    try {
        for (; r < size; r++)//遍历集合A
            //判断集合B中是否包含集合A中的当前元素
            if (c.contains(elementData[r]) == complement)
                //如果包含则直接保存到集合A(w是从0位置开始的)。
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        //如果contains方法使用过程报异常
        if (r != size) {
            //复制剩余的元素
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            //w为当前集合A的length(前面try已经复制了w个了，后面size-r个还没对比就发生了异常)
            w += size - r;

        }
        //如果集合A的大小放发生改变
        if (w != size) {
            // clear to let GC do its work
            //这里有两个用途，在removeAll()时，w一直为0，就直接跟clear一样，全是为null。
            //retainAll()：没有一个交集返回true，有交集但不全交也返回true，而两个集合相等的时候，返回false，
            // 所以不能根据返回值来确认两个集合是否有交集，而是通过原集合的大小是否发生改变来判断，
            // 如果原集合中还有元素即w!=size，则代表有交集，而原集合没有元素了即w==size，说明两个集合没有交集。
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;//上一行size-w个置空了
            size = w;//当前size就是w个
            modified = true;
        }
    }
    return modified;
}
```

## 6.总结
- arrayList可以存放null。
- arrayList本质上就是一个elementData数组。
- arrayList区别于数组的地方在于能够自动扩展大小，其中关键的方法就是gorw()方法。
- arrayList中removeAll(collection c)和clear()的区别就是removeAll可以删除批量指定的元素，而clear是全是删除集合中的元素。
- arrayList由于本质是数组，所以它在数据的查询方面会很快，而在插入删除这些方面，性能下降很多，有移动很多数据才能达到应有的效果
- arrayList实现了RandomAccess，所以在遍历它的时候推荐使用for循环。




