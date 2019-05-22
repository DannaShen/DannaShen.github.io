---
layout: post
title: synchronized同步方法
categories: 多线程
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1.线程安全与非线程安全
#### 1.1 方法内的变量为线程安全
“非线程安全”问题存在于"实例变量"中，如果是方法内部的私有变量， 则不存在“非线程安全”问题，所得结果也就是"线程安全" 的了。
下面的示例就是方法内部声明一个变量时，是不存在"非线程安全"问题的。
``` java
public class HasSelfPrivateNum {

    public void addI(String username){
        try {
        int num;//方法内部变量
        if (username.equals("a")){
            num=100;
            System.out.println("a set over!");
                Thread.sleep(2000);
        }else {
            num=200;
            System.out.println("b set over!");
        }
            System.out.println(username+" num="+num);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
public class ThreadA extends Thread{

    private HasSelfPrivateNum numRef;
    public ThreadA(HasSelfPrivateNum numRef){
        super();
        this.numRef=numRef;
    }
    @Override
    public void run() {
        super.run();
        numRef.addI("a");
    }
}
public class ThreadB extends Thread {

    private HasSelfPrivateNum numRef;
    public ThreadB(HasSelfPrivateNum numRef){
        super();
        this.numRef=numRef;
    }
    @Override
    public void run() {
        super.run();
        numRef.addI("b");
    }
}
public static void main(String[] args) {
        HasSelfPrivateNum numRef=new HasSelfPrivateNum();
        ThreadA athread=new ThreadA(numRef);
        athread.start();
        ThreadB bthread=new ThreadB(numRef);
        bthread.start();
    }
```
结果：
```
a set over!
b set over!
a num=100
b num=200
```
可见，方法中的变量不存在非线程安全问题，永远都是线程安全的。这是方法内部的变量是线程私有的特性造成的。
#### 1.2 实例变量非线程安全
如果多个线程共同访问1个对象中的实例变量，则有可能出现"非线程安全"问题。线程访问的对象中如果有多个实例变量，则运行的结果
有可能出现交叉的情况。如果对象仅有l 个实例变量，则有可能出现覆盖的情况。  
``` java
public class HasSelfPrivateNum {

    int num;//方法外部变量
    public void addI(String username){
        try {
        if (username.equals("a")){
            num=100;
            System.out.println("a set over!");
                Thread.sleep(2000);
        }else {
            num=200;
            System.out.println("b set over!");
        }
            System.out.println(username+" num="+num);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
其他代码如上。这时，如果想要线程安全只需要在public yoid addl(String useπlame)方法前加关键字synchronized即可，由于是同步访问，
所以先打印出a ，然后打印出b，结果如下：  
```
a set over!
a num=100
b set over!
b num=200
```
#### 1.3 多个对象多个锁
``` java
public static void main(String[] args) {
        HasSelfPrivateNum numRef1=new HasSelfPrivateNum();
        HasSelfPrivateNum numRef2=new HasSelfPrivateNum();
        ThreadA athread=new ThreadA(numRef1);
        athread.start();
        ThreadB bthread=new ThreadB(numRef2);
        bthread.start();
    }
```
结果：  
```
a set over!
b set over!
b num=200
a num=100
```
上面示例是两个线程分别访问同一个类的两个不同实例的相同名称的同步方法，效果却是以异步的方式运行的。本示例由于创建了2个业务对象，
在系统中产生2个锁，所以结果是异步的。
