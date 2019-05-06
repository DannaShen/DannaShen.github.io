---
layout: post
title: 序列化、反序列化、浅拷贝、深拷贝
categories: Java基础
description: 序列化和反序列化，serialVersionUID的作用及Externalizable，浅拷贝和深拷贝区别详细分析
keywords: 序列化、反序列化、serialVersionUID、Externalizable、浅拷贝、深拷贝
---


## 一、序列化和反序列化

### 1.概念和用途
1) 概念  
序列化：把对象转换为字节序列的过程    
反序列化：把字节序列恢复为对象的过程  
2) 对象序列化的用途  
- 把对象的字节序列永久的保存到硬盘上，通常放到一个文件。
比如：最常见的是Web服务器中的Session对象，当有10万用户并发访问，就有可能出现10万个Session对象，内存可能吃不消，
于是Web容器就会把一些seesion先序列化到硬盘中，等要用了，再把保存在硬盘中的对象还原到内存中。  
- 在网络上传送对象的字节序列
比如：当两个进程在进行远程通信时，彼此可以发送各种类型的数据。无论是何种类型的数据，都会以二进制序列的形式在网络上传送。
发送方需要把这个Java对象转换为字节序列，才能在网络上传送；接收方则需要把字节序列再恢复为Java对象。

### 2.步骤
（1）序列化的步骤
``` java
    Person person = new Person();
    person.setName("gacl");
    person.setAge(25);
    person.setSex("男");
    // ObjectOutputStream 对象输出流，将Person对象存储到E盘的Person.txt文件中，完成对Person对象的序列化操作
    ObjectOutputStream oo = new ObjectOutputStream(new FileOutputStream(
            new File("E:/Person.txt")));
    oo.writeObject(person);
    System.out.println("Person对象序列化成功！");
    oo.close();
```
（2）反序列化步骤
``` java
    ObjectInputStream ois = new ObjectInputStream(new FileInputStream(
                new File("E:/Person.txt")));
    Person person = (Person) ois.readObject();
    System.out.println("Person对象反序列化成功！");
```

## 二、serialVersionUID的作用

### 1.概念
即序列化的版本号，凡是实现了Serializable接口的类都有一个表示序列化版本的标识符静态变量
``` java
    private static final long serialVersionUID
```
### 生成方式
（1）默认的
``` java
    private static final long serialVersionUID = 1L;
```
（2）根据类名、接口名、方法和属性来生成
``` java
    private static final long serialVersionUID = 4603642343377807741L;
```

### 作用
定义一个类，没有定义serialVersionUID。当对该类序列化，然后再反序列化。再次对这个类修改，然后执行反序列化会报错：InvalidClassException  
这是因为文件流中的class和classpath中的class(修改过后的class)，不兼容了，处于安全机制考虑，程序抛出了错误。由于没有显指定 serialVersionUID，
添加了一个字段后，编译器又为我们生成了一个UID，当然和前面保存在文件中的那个不一样了，于是就出现了2个序列化版本号不一致的错误，所以需要自己去指定serialVersionUID。

## 三、Serializable和Externalizable区别

### 1.概念
被Serializable接口声明的类的对象的内容都将被序列化，如果希望自己指定序列化的内容，则可以让一个类实现Externalizable接口
### 2.步骤
``` java
    public class Person implements Externalizable  {
        .
        .
        .
        // 覆写此方法，根据需要读取内容，反序列化时使用  
        public void readExternal(ObjectInput in)  throws IOException,  
                ClassNotFoundException {  
            this.name = (String)in.readObject() ;  // 读取姓名属性  
            this.age = in.readInt() ;              // 读取年龄  
        }  
        // 覆写此方法，根据需要可以保存属性或具体内容， 序列化时使用  
        public void writeExternal(ObjectOutput out)  throws IOException {  
            out.writeObject(this.name) ;          // 保存姓名属性  
            out.writeInt(this.age) ;              // 保存年龄属性  
        }  
    }  
```
### 3.区别

| 序号 | 区别     |Serializable                |Externalizable|
| --- |-------|-------                      |---|
| 1 |实现复杂度|实现简单，Java对其有默认支持    |实现复杂，由开发人员自己完成|
| 2 |执行效率  |所有对象由Java统一保存，性能较低|开发人员决定哪个对象保存，可能造成速度提升|
| 3 |保存信息  |保存时占用空间大               |部分存储，可能造成空间减少|

## 四、深拷贝和浅拷贝详解
### 1.拷贝的引入

（1）引用拷贝  

代码：
``` java
    Teacher teacher = new Teacher("Taylor",26);
    Teacher otherteacher = teacher;
    System.out.println(teacher);
    System.out.println(otherteacher);
```
输出结果：

    blog.Teacher@355da254
    blog.Teacher@355da254

图解：
![](/images/posts/Java基础/引用拷贝.png) 
（2）对象拷贝  

代码：
``` java
    Teacher teacher = new Teacher("Swift",26);
    Teacher otherteacher = (Teacher)teacher.clone();
    System.out.println(teacher);
    System.out.println(otherteacher);
```
输出结果：

    blog.Teacher@355da254
    blog.Teacher@4dc63996
    
图解：
![](/images/posts/Java基础/对象拷贝.png) 
**注：深拷贝和浅拷贝都是对象拷贝**  
（3）拷贝实现
让实体类实现cloneable接口，并重写clone方法。  
- clone方法是定义在object中，并且是projected(同package下的其他类中访问;子类可以访问)修饰。    
但是cloneable接口是空的，那为什么还要实现该接口呢？因为该接口是一个标记接口，只有实现这个接口后，然后在类中重写Object中的clone方法，
然后通过类调用clone方法才能克隆成功，只有如果不实现该接口，会报错CloneNotSupportedException。  
- 但是如何判断类是否实现了cloneable接口了呢？原因在于这个方法中有一个native关键字，native修饰的方法都是空的方法，但是这些方法都是有实现体的
（这里也就间接说明了native关键字不能与abstract同时使用），只不过native方法调用的实现体，都是非java代码编写的（例如：调用的是在jvm中编写的C的接口），
每一个native方法在jvm中都有一个同名的实现体，native方法在逻辑上的判断都是由实现体实现的，另外这种native修饰的方法对返回类型，异常控制等都没有约束。 

### 2.浅拷贝
（1）定义： 
比如  
``` java
    Student stu2=(Student)stu1.clone();   
```
被复制得到的对象(stu2)的所有变量都含有与原来的对象(stu1)相同的值，而stu2所有的对其他对象的引用(其他对象的引用:类中定义的变量是引用型)仍然指向原来的对象。
即对象的浅拷贝会对“主”对象进行拷贝，但不会复制主对象里面的对象。”里面的对象“会在原来的对象和它的副本之间共享。  

简而言之，浅拷贝仅仅复制当前对象，而不复制它所引用的对象。  
（2）拷贝实例：
``` java
    Teacher teacher = new Teacher();
    teacher.setName("Delacey");
    teacher.setAge(29);

    Student2 student1 = new Student2();
    student1.setName("Dream");
    student1.setAge(18);
    student1.setTeacher(teacher);

    Student2 student2 = (Student2) student1.clone();
    System.out.println("拷贝后");
    System.out.println(student2.getName());
    System.out.println(student2.getAge());
    System.out.println(student2.getTeacher().getName());
    System.out.println(student2.getTeacher().getAge());
    System.out.println("修改老师的信息后-------------");

    // 修改老师的信息
    teacher.setName("Jam");
    System.out.println(student1.getTeacher().getName());
    System.out.println(student2.getTeacher().getName());
```
``` java
    class Teacher implements Cloneable
    {
        private String name;
        private int age;
    
        public String getName()
        {
            return name;
        }
        public void setName(String name)
        {
            this.name = name;
        }
        public int getAge()
        {
            return age;
        }
        public void setAge(int age)
        {
            this.age = age;
        }
    }
```
``` java
    class Student implements Cloneable
    {
        private String name;
        private int age;
        private Teacher teacher;
    
        public String getName()
        {
            return name;
        }
        public void setName(String name)
        {
            this.name = name;
        }
        public int getAge()
        {
            return age;
        }
        public void setAge(int age)
        {
            this.age = age;
        }
        public Teacher getTeacher()
        {
            return teacher;
        }
        public void setTeacher(Teacher teacher)
        {
            this.teacher = teacher;
        }
        @Override
        public Object clone() throws CloneNotSupportedException
        {
            Object object = super.clone();
            return object;
        }
    }
```
输出结果为：  

    拷贝后
    Dream
    18
    Delacey
    29
    修改老师的信息后-------------
    Jam
    Jam
    
（3）图解  
![](/images/posts/Java基础/浅拷贝.png) 

### 3.深拷贝
（1）定义：  
深拷贝是一个整个独立的对象拷贝，深拷贝会拷贝所有的属性,并拷贝属性指向的动态分配的内存。当对象和它所引用的对象一起拷贝时即发生深拷贝。
深拷贝相比于浅拷贝速度较慢并且花销较大。  

简而言之，深拷贝把要复制的对象所引用的对象都复制了一遍。  
（2）拷贝实例：  
``` java
    Teacher2 teacher = new Teacher2();
    teacher.setName("Delacey");
    teacher.setAge(29);

    Student3 student1 = new Student3();
    student1.setName("Dream");
    student1.setAge(18);
    student1.setTeacher(teacher);

    Student3 student2 = (Student3) student1.clone();
    System.out.println("拷贝后");
    System.out.println(student2.getName());
    System.out.println(student2.getAge());
    System.out.println(student2.getTeacher().getName());
    System.out.println(student2.getTeacher().getAge());
    System.out.println("修改老师的信息后-------------");

    // 修改老师的信息
    teacher.setName("Jam");
    System.out.println(student1.getTeacher().getName());
    System.out.println(student2.getTeacher().getName());
```
``` java
    class Teacher implements Cloneable {
        private String name;
        private int age;
    
        public String getName()
        {
            return name;
        }
        public void setName(String name)
        {
            this.name = name;
        }
        public int getAge()
        {
            return age;
        }
        public void setAge(int age)
        {
            this.age = age;
        }
        @Override
        public Object clone() throws CloneNotSupportedException
        {
            return super.clone();
        }
    }
```
``` java
    class Student implements Cloneable {
        private String name;
        private int age;
        private Teacher teacher;
    
        public String getName()
        {
            return name;
        }
        public void setName(String name)
        {
            this.name = name;
        }
        public int getAge()
        {
            return age;
        }
        public void setAge(int age)
        {
            this.age = age;
        }
        public Teacher2 getTeacher()
        {
            return teacher;
        }
        public void setTeacher(Teacher2 teacher)
        {
            this.teacher = teacher;
        }
        @Override
        public Object clone() throws CloneNotSupportedException
        {
            // 浅复制时：
            // Object object = super.clone();
            // return object;
    
            // 改为深复制：
            Student student = (Student) super.clone();
            // 本来是浅复制，现在将Teacher对象复制一份并重新set进来
            student.setTeacher((Teacher) student.getTeacher().clone());
            return student;
        }
    
    }

```
（3）图解：  
![](/images/posts/Java基础/深拷贝.png) 















