---
layout: post
title: spring之面向切面
categories: spring
description: spring之面向切面编程
keywords: spring,切面,aop
---

> 1.在软件开发中，散布于应用中多处的功能被称为横切关注点（cross-cutting concern）。  
> 2.通常来讲，这些横切关注点从概念上是与应用的业务逻辑相分离的（但是往往会直接嵌入到应用的业务逻辑之中）。  
> 3.把这些横切关注点与业务逻辑相分离正是面向切面编程（AOP）所要解决的问题。  
> 4.本章展示了Spring对切面的支持，包括如何把普通类声明为一个切面和如何使用注解创建切面。除此之外，我们还会看到AspectJ——另
  一种流行的AOP实现

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 1.什么是面向切面编程
&emsp;&emsp;横切关注点可以被描述为影响应用多处的功能。例如，安全就是一个横切关注点，应用中的许多方法都会涉及到安全规则。  
&emsp;&emsp;图4.1直观呈现了横切关注点的概念。  
![](/images/posts/spring/spring-aop/crosscuting_concerns_modularize.png)  
&emsp;&emsp;图4.1展现了一个被划分为模块的典型应用。  
每个模块的核心功能都是为特定业务领域提供服务，但是这些模块都需要类似的辅助功能，
例如安全和事务管理。如果要重用通用功能的话，最常见的面向对象技术是继承（inheritance）或委托（delegation）。但是，
如果在整个应用中都使用相同的基类，继承往往会导致一个脆弱的对象体系；而使用委托可能需要对委托对象进行复杂的调用。  
&emsp;&emsp;切面提供了取代继承和委托的另一种可选方案，而且在很多场景下更清晰简洁。在使用面向切面编程时，我们仍然在一个地方定义通用功能，
但是可以通过声明的方式定义这个功能要以何种方式在何处应用，而无需修改受影响的类。
横切关注点可以被模块化为特殊的类，这些类被称为切面（aspect）。这样做有两个好处：首先，现在每个关注点都集中于一个地方，
而不是分散到多处代码中；其次，服务模块更简洁，因为它们只包含主要关注点（或核心功能）的代码，而次要关注点的代码被转移到切面中了。

#### 1.1 定义AOP术语
&emsp;&emsp;描述切面的常用术语有通知（advice）、切点（pointcut）和连接点（join point）。  
&emsp;&emsp;图4.2展示了这些概念是如何关联在一起的。  
![](/images/posts/spring/spring-aop/aop_relation_graph.png)  

###### 1.1.1 通知（Advice）

&emsp;&emsp;当抄表员出现在我们家门口时，他们要登记用电量并回去向电力公司报告。显然，他们必须有一份需要抄表的住户清单，他们所汇报的信
息也很重要，但记录用电量才是抄表员的主要工作。  
&emsp;&emsp;切面也有目标——它必须要完成的工作。在AOP术语中，切面的工作被称为通知。  
&emsp;&emsp;通知定义了切面是什么以及何时使用。除了描述切面要完成的工作，通知还解决了何时执行这个工作的问题。它应该应用在某个方法被调
用之前？之后？之前和之后都调用？还是只在方法抛出异常时调用  
> Spring切面可以应用5种类型的通知：
> 前置通知（Before）：在目标方法被调用之前调用通知功能；  
> 后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；  
> 返回通知（After-returning）：在目标方法成功执行之后调用通知；  
> 异常通知（After-throwing）：在目标方法抛出异常后调用通知；  
> 环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为。  

###### 1.1.2 连接点（Join point）
&emsp;&emsp;电力公司为多个住户提供服务，甚至可能是整个城市。每家都有一个电表，这些电表上的数字都需要读取，因此每家都是抄表员的潜在目标。
抄表员也许能够读取各种类型的设备，但是为了完成他的工作，他的目标应该房屋内所安装的电表。  
&emsp;&emsp;同样，我们的应用可能也有数以千计的时机来应用通知。这些<u>时机被称为连接点</u>。连接点是在应用执行过程中能够插入切面的一个点。这个
点可以是调用方法时、抛出异常时、甚至修改一个字段时。切面代码可以利用这些点插入到应用的正常流程之中，并添加新的行为。
（网上关于切点和连接点的定义回答:切点的定义会匹配通知所要织入的一个或多个连接点，我们通常会在类、方法等地方定义切点，
而连接点就会在这些切点的某一个时机，比如方法执行前、执行后进行连接，应用通知。）
###### 1.1.3 切点（Poincut）
&emsp;&emsp;如果让一位抄表员访问电力公司所服务的所有住户，那肯定是不现实的。实际上，电力公司为每一个抄表员都分别指定某一块区域的住户。
类似地，一个切面并不需要通知应用的所有连接点。切点有助于缩小切面所通知的连接点的范围。
如果说通知定义了切面的“什么”和“何时”的话，那么切点就定义了“何处”。切点的定义会匹配通知所要织入的一个或多个连接点。  
&emsp;&emsp;我们通常使用明确的类和方法名称，或是利用正则表达式定义所匹配的类和方法名称来指定这些切点。有些AOP框架允许我们创建动态的切
点，可以根据运行时的决策（比如方法的参数值）来决定是否应用通知。
###### 1.1.4 切面（Aspect）
&emsp;&emsp;当抄表员开始一天的工作时，他知道自己要做的事情（报告用电量）和从哪些房屋收集信息。因此，他知道要完成工作所需要的一切东西。  
&emsp;&emsp;切面是通知和切点的结合。通知和切点共同定义了切面的全部内容——它是什么，在何时和何处完成其功能。
###### 1.1.6 引入（Introduction）
 &emsp;&emsp;引入允许我们向现有的类添加新方法或属性。例如，我们可以创建一个Auditable通知类，该类记录要对象最后一次修改时的状态。
 这很简单，只需一个方法，setLastModified(Date)，和一个实例变量来保存这个状态。然后，这个新方法和实例变量就可以被引入到
 现有的类中，从而可以在无需修改这些现有的类的情况下，让它们具有新的行为和状态。
###### 1.1.7 织入（Weaving）
 &emsp;&emsp;织入是把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。在目标对象的生命周期里有多个点
 可以进行织入：  
 1. 编译期：切面在目标类编译时被织入。这种方式需要特殊的编译器。AspectJ的织入编译器就是以这种方式织入切面的。
 2. 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特殊的类加载器（ClassLoader），它可以在目标类被引入应用
 之前增强该目标类的字节码。AspectJ 5的加载时织入（load-timeweaving，LTW）就支持以这种方式织入切面。
 3. 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。
 Spring AOP就是以这种方式织入切面的。  
 &emsp;&emsp;再看一下图4.1，现在我们已经了解了如下的知识，  
 通知包含了需要用于多个应用对象的横切行为；  
 连接点是程序执行过程中能够应用通知的所有点；    
 切点定义了通知被应用的具体位置（在哪些连接点）。其中关键的概念是切点定义了哪些连接点会得到通知。
 
#### 1.2 Spring对AOP的支持

&emsp;&emsp;不是所有的AOP框架都是相同的，它们在连接点模型上可能有强弱之分。有些允许在字段修饰符级别应用通知，而另一些只支持与方法
调用相关的连接点。它们织入切面的方式和时机也有所不同。但是无论如何，创建切点来定义切面，在指定的连接点把切面织入是AOP框架的基本功
能。  
> Spring提供了4种类型的AOP支持：    
> 基于代理的经典Spring AOP；  
> 纯POJO切面；  
> @AspectJ注解驱动的切面；  
> 注入式AspectJ切面（适用于Spring各版本）。  

&emsp;&emsp;前三种都是Spring AOP实现的变体，pring SAOP构建在动态代理基础之上，因此，Spring对AOP的支持局限于方法拦截。  
&emsp;&emsp;Spring的经典AOP编程模型并不怎么样。当然，曾经它的确非常棒。但是现在Spring提供了更简洁和干净的面向切面编程方式。
引入了简单的声明式AOP和基于注解的AOP之后，Spring经典的AOP看起来就显得非常笨重和过于复杂，
直接使用ProxyFactory Bean会让人感觉厌烦。所以在本书中我不会再介绍经典的Spring AOP。  
&emsp;&emsp;借助Spring的aop命名空间，我们可以将纯POJO转换为切面。实际上，这些POJO只是提供了满足切点条件时所要调用的方法。
遗憾的，这种技是术需要XML配置，但这的确是声明式地将对象转换为切面的简便方式。  
&emsp;&emsp;Spring借鉴了AspectJ的切面，以提供注解驱动的AOP。本质上，它依然是Spring基于代理的AOP，但是编程模型几乎与编写成熟的AspectJ
注解切面完全一致。这种AOP风格的好处在于能够不使用XML来完成功能。  
&emsp;&emsp;如果你的AOP需求超过了简单的方法调用（如构造器或属性拦截），那么你需要考虑使用AspectJ来实现切面。在这种情况下，上文所示
的第四种类型能够帮助你将值注入到AspectJ驱动的切面中。  
我们在将在本章展示更多的Spring AOP技术，但是在开始之前，我们必须要了解Spring AOP框架的一些关键知识。  
###### 1.2.1 Spring通知是Java编写的
&emsp;&emsp;Spring所创建的通知都是用标准的Java类编写的。这样的话，我们就可以使用与普通Java开发一样的集成开发环境（IDE）来开发切面。
而且，定义通知所应用的切点通常会使用注解或在Spring配置文件里采用XML来编写，这两种语法对于Java开发者来说都是相当熟悉的。  
&emsp;&emsp;AspectJ与之相反。虽然AspectJ现在支持基于注解的切面，但AspectJ最初是以Java语言扩展的方式实现的。这种方式有优点也有缺点。通
过特有的AOP语言，我们可以获得更强大和细粒度的控制，以及更丰富的AOP工具集，但是我们需要额外学习新的工具和语法。
###### 1.2.2 Spring在运行时通知对象
&emsp;&emsp;通过在代理类中包裹切面，Spring在运行期把切面织入到Spring管理的bean中。如图4.3所示，代理类封装了目标类，并拦截被通知方法的
调用，再把调用转发给真正的目标bean。当代理拦截到方法调用时，在调用目标bean方法之前，会执行切面逻辑。
![](/images/posts/spring/spring-aop/aop_proxy.png)  
&emsp;&emsp;直到应用需要被代理的bean时，Spring才创建代理对象。如果使用的是ApplicationContext的话，在ApplicationContext从
BeanFactory中加载所有bean的时候，Spring才会创建被代理的对象。因为Spring运行时才创建代理对象，所以我们不需要特殊的编译
器来织入Spring AOP的切面。
###### 1.2.3 Spring只支持方法级别的连接点
&emsp;&emsp;正如前面所探讨过的，通过使用各种AOP方案可以支持多种连接点模型。因为Spring基于动态代理，所以Spring只支持方法连接点。
这与一些其他的AOP框架是不同的，例如AspectJ和JBoss，除了方法切点，它们还提供了字段和构造器接入点。Spring缺少对字段连接点的
支持，无法让我们创建细粒度的通知，例如拦截对象字段的修改。而且它不支持构造器连接点，我们就无法在bean创建时应用通知。  
但是方法拦截可以满足绝大部分的需求。如果需要方法拦截之外的连接点拦截功能，那么我们可以利用Aspect来补充Spring AOP的功能。  
&emsp;&emsp;对于什么是AOP以及Spring如何支持AOP的，我们现在已经有了一个大致的了解。现在是时候学习如何在Spring中创建切面了，让我们先
从Spring的声明式AOP模型开始。
## 2 通过切点来选择连接点
&emsp;&emsp;正如之前所提过的，切点用于准确定位应该在什么地方应用切面的通知。通知和切点是切面的最基本元素。因此，了解如何编写切点非常重要  
&emsp;&emsp;在Spring AOP中，要使用AspectJ的切点表达式语言来定义切点。如果你已经很熟悉AspectJ，那么在Spring中定义切点就感觉非常自然。但
是如果你一点都不了解AspectJ的话，本小节我们将快速介绍一下如何编写AspectJ风格的切点。  
&emsp;&emsp;关于Spring AOP的AspectJ切点，最重要的一点就是Spring仅支持AspectJ切点指示器（pointcut designator）的一个子集。让我们回顾
下，Spring是基于代理的，而某些切点表达式是与基于代理的AOP无关的。表4.1列出了Spring AOP所支持的AspectJ切点指示器。
![](/images/posts/spring/spring-aop/Aspectj_indicator.png)  
&emsp;&emsp;在Spring中尝试使用AspectJ其他指示器时，将会抛出IllegalArgument-Exception异常。  
当我们查看如上所展示的这些Spring支持的指示器时，注意只有execution指示器是实际执行匹配的，而其他的指示器都是用来限制匹配的。
这说明execution指示器是我们在编写切点定义时最主要使用的指示器。在此基础上，我们使用其他指示器来限制所匹配的切点。
#### 2.1 编写切点
&emsp;&emsp;为了阐述Spring中的切面，我们需要有个主题来定义切面的切点。为此，我们定义一个Performance接口：
``` java
public interface Performance {
    public void perform();
}

```
&emsp;&emsp;Performance可以代表任何类型的现场表演，如舞台剧、电影或音乐会。假设我们想编写Performance的perform()方法触发的通知。
图4.4展现了一个切点表达式，这个表达式能够设置当perform()方法执行时触发通知的调用。
![](/images/posts/spring/spring-aop/pointcut_expression.png)    
&emsp;&emsp;我们使用execution()指示器选择Performance的perform()方法。方法表达式以“*”号开始，表明了我们不关心方法返回值的类
型。然后，我们指定了全限定类名和方法名。对于方法参数列表，我们使用两个点号（..）表明切点要选择任意的perform()方法，无
论该方法的入参是什么。  
&emsp;&emsp;现在假设我们需要配置的切点仅匹配concert包。在此场景下，可以使用within()指示器来限制匹配，如图4.5所示。  
![](/images/posts/spring/spring-aop/pointcut_use_limit.png)  
&emsp;&emsp;请注意我们使用了“&&”操作符把execution()和within()指示器连接在一起形成与（and）关系（切点必须匹配所有的指示器）。类
似地，我们可以使用“||”操作符来标识或（or）关系，而使用“!”操作符来标识非（not）操作。    
&emsp;&emsp;因为“&”在XML中有特殊含义，所以在Spring的XML配置里面描述切点时，我们可以使用and来代替“&&”。同样，or和not可以分别用来
代替“||”和“!”。  
#### 2.2 在切点中选择bean
&emsp;&emsp;除了表4.1所列的指示器外，Spring还引入了一个新的bean()指示器，它允许我们在切点表达式中使用bean的ID来标识bean。bean()
使用bean ID或bean名称作为参数来限制切点只匹配特定的bean。例如，考虑如下的切点：  
``` java
execution(* concert.Performance.perform()) and bean('woodstock')  
```  
&emsp;&emsp;在这里，我们希望在执行Performance的perform()方法时应用通知，但限定bean的ID为woodstock。  
在某些场景下，限定切点为指定的bean或许很有意义，但我们还可以使用非操作为除了特定ID以外的其他bean应用通知：  
``` java
execution(* concert.Performance.perform()) and !bean('woodstock')   
```
&emsp;&emsp;在此场景下，切面的通知会被编织到所有ID不为woodstock的bean中。  
&emsp;&emsp;现在，我们已经讲解了编写切点的基础知识，让我们再了解一下如何编写通知和使用这些切点声明切面。
## 3 使用注解创建切面
&emsp;&emsp;使用注解来创建切面是AspectJ 5所引入的关键特性。AspectJ 5之前，编写AspectJ切面需要学习一种Java语言的扩展，但是AspectJ面向注解
的模型可以非常简便地通过少量注解把任意类转变为切面。我们已经定义了Performance接口，它是切面中切点的目标对象。
现在，让我们使用AspecJ注解来定义切面。
#### 3.1 定义切面
> 如果一场演出没有观众的话，那不能称之为演出。对不对？从演出的角度来看，观众是非常重要的，但是对演出本身的功能来讲，
它并不是核心，这是一个单独的关注点。因此，将观众定义为一个切面，并将其应用到演出上就是较为明智的做法。

&emsp;&emsp;程序清单4.1展现了Audience类，它定义了我们所需的一个切面。  
&emsp;&emsp;程序清单4.1　Audience类：观看演出的切面  
``` java

@Aspect
public class Audience {

    @Before("execution(**concert.Performance.perform(..))")
    public void silenceCellPhones(){
        System.out.println("Silencing  cell phone");
    }
    @Before("execution(**concert.Performance.perform(..))")
    public void takeSeats(){
        System.out.println("Taking seats");
    }

    @AfterReturning("execution(**concert.Performance.perform(..))")
    public void applause(){
        System.out.println("CLAP CLAP CLAP!!!");
    }

    @AfterThrowing("execution(**concert.Performance.perform(..))")
    public void demandRefund(){
        System.out.println("Demanding a refund");
    }
}	
```
&emsp;&emsp;Audience类使用@AspectJ注解进行了标注。该注解表明Audience不仅仅是一个POJO，还是一个切面。Audience类中的方法
都使用注解来定义切面的具体行为。  
&emsp;&emsp;Audience有四个方法，定义了一个观众在观看演出时可能会做的事情。在演出之前，观众要就坐（takeSeats()）并将手机调至
静音状态（silenceCellPhones()）。如果演出很精彩的话，观众应该会鼓掌喝彩（applause()）。不过，如果演出没有达到观众预期的话，
观众会要求退款（demandRefund()）。  
可以看到，这些方法都使用了通知注解来表明它们应该在什么时候调用。AspectJ提供了五个注解来定义通知，如表4.2所示。
![](/images/posts/spring/spring-aop/methods_of_Aspectj_annotaion_desifine.png)  
&emsp;&emsp;Audience使用到了前面五个注解中的三个。takeSeats()和silence CellPhones()方法都用到了@Before注解，表明它们应
该在演出开始之前调用。applause()方法使用了@AfterReturning注解，它会在演出成功返回后调用。
demandRefund()方法上添加了@AfterThrowing注解，这表明它会在抛出异常以后执行。
你可能已经注意到了，所有的这些注解都给定了一个切点表达式作为它的值，同时，这四个方法的切点表达式都是相同的。其实，它们可
以设置成不同的切点表达式，但是在这里，这个切点表达式就能满足所有通知方法的需求。让我们近距离看一下这个设置给通知注解的切
点表达式，我们发现它会在Performance的perform()方法执行时触发。  
&emsp;&emsp;相同的切点表达式我们重复了四遍，这可真不是什么光彩的事情。这样的重复让人感觉有些不对劲。如果我们只定义这个切点一次，然后
每次需要的时候引用它，那么这会是一个很好的方案。  
&emsp;&emsp;幸好，我们完全可以这样做：@Pointcut注解能够在一个@AspectJ切面内定义可重用的切点。接下来的程序清单4.2展现了新的Audience，
现在它使用了@Pointcut。  
&emsp;&emsp;程序清单4.2　通过@Pointcut注解声明频繁使用的切点表达式  
``` java
@Aspect
public class Audience {

    @Pointcut("execution(**concert.Performance.perform(..))")
    public  void  performance(){}

    @Before("performance()")
    public void silenceCellPhones(){
        System.out.println("Silencing  cell phone");
    }
    @Before("performance()")
    public void takeSeats(){
        System.out.println("Taking seats");
    }

    @AfterReturning("performance()")
    public void applause(){
        System.out.println("CLAP CLAP CLAP!!!");
    }

    @AfterThrowing("performance()")
    public void demandRefund(){
        System.out.println("Demanding a refund");
    }
}
```
&emsp;&emsp;在Audience中，performance()方法使用了@Pointcut注解。为@Pointcut注解设置的值是一个切点表达式，就像之前在通知注解上所设置的那样。
通过在performance()方法上添加@Pointcut注解，我们实际上扩展了切点表达式语言，这样就可以在任何的切点表达式中使用performance()了，
如果不这样做的话，你需要在这些地方使用那个更长的切点表达式。我们现在把所有通知注解中的长表达式都替换成了performance()。  
&emsp;&emsp;performance()方法的实际内容并不重要，在这里它实际上应该是空的。其实该方法本身只是一个标识，供@Pointcut注解依附。  
&emsp;&emsp;需要注意的是，除了注解和没有实际操作的performance()方法，Audience类依然是一个POJO。我们能够像使用其他的Java类那样调用它的方法，
它的方法也能够独立地进行单元测试，这与其他的Java类并没有什么区别。Audience只是一个Java类，只不过它通过注解表明会作为切面使用而已。  
像其他的Java类一样，它可以装配为Spring中的bean：
``` java
    @Bean
    public Audience audience(){
        return  new Audience();
    }
```
&emsp;&emsp;如果你就此止步的话，Audience只会是Spring容器中的一个bean。即便使用了AspectJ注解，但它并不会被视为切面，这些注解不会解析，
也不会创建将其转换为切面的代理。  
如果你使用JavaConfig的话，可以在配置类的类级别上通过使用EnableAspectJ-AutoProxy注解启用自动代理功能。  
&emsp;&emsp;程序清单4.3展现了如何在JavaConfig中启用自动代理。
``` java
@Configuration
@ComponentScan
@EnableAspectJAutoProxy
public class ConcertConfig {

    @Bean
    public Audience audience(){
        return new Audience();
    }
}
```
&emsp;&emsp;假如你在Spring中要使用XML来装配bean的话，那么需要使用Springaop命名空间中的``` java <aop:aspectj-autoproxy> ```元素。  
不管你是使用JavaConfig还是XML，AspectJ自动代理都会为使用@Aspect注解的bean创建一个代理，这个代理会围绕着所有该切面
的切点所匹配的bean。在这种情况下，将会为Concertbean创建一个代理，Audience类中的通知方法将会在perform()调用前后执行。    
&emsp;&emsp;我们需要记住的是，Spring的AspectJ自动代理仅仅使用@AspectJ作为创建切面的指导，切面依然是基于代理的。在本质上，它依然是
Spring基于代理的切面。这一点非常重要，因为这意味着尽管使用的是@AspectJ注解，但我们仍然限于代理方法的调用。如果想利用
AspectJ的所有能力，我们必须在运行时使用AspectJ并且不依赖Spring来创建基于代理的切面。    
&emsp;&emsp;到现在为止，我们的切面在定义时，使用了不同的通知方法来实现前置通知和后置通知。但是表4.2还提到了另外的一种通知：环绕通知
（around advice）。环绕通知与其他类型的通知有所不同，因此值得花点时间来介绍如何进行编写。
#### 3.2 创建环绕通知
&emsp;&emsp;环绕通知是最为强大的通知类型。它能够让你所编写的逻辑将被通知的目标方法完全包装起来。实际上就像在一个通知方法中同时编写前
置通知和后置通知。  
&emsp;&emsp;为了阐述环绕通知，我们重写Audience切面。这次，我们使用一个环绕通知来代替之前多个不同的前置通知和后置通知。  
&emsp;&emsp;程序清单4.5　使用环绕通知重新实现Audience切面
``` java

@Aspect
public class Audience {

    @Pointcut("execution(**concert.Performance.perform(..))")
    public  void  performance(){}

    @Around("performance()")
    public void watchPerformance(ProceedingJoinPoint jp){
        try {
            System.out.println("Silencing  cell phone");
            System.out.println("Taking seats");
            jp.proceed();
            System.out.println("CLAP CLAP CLAP!!!");
        } catch (Throwable throwable) {
            System.out.println("Demanding a refund");
        }
    }

}
```
&emsp;&emsp;在这里，@Around注解表明watchPerformance()方法会作为performance()切点的环绕通知。在这个通知中，观众在演出之
前会将手机调至静音并就坐，演出结束后会鼓掌喝彩。像前面一样，如果演出失败的话，观众会要求退款。  
&emsp;&emsp;可以看到，这个通知所达到的效果与之前的前置通知和后置通知是一样的。但是，现在它们位于同一个方法中，不像之前那样分散在四个
不同的通知方法里面。  
&emsp;&emsp;关于这个新的通知方法，你首先注意到的可能是它接受ProceedingJoinPoint作为参数。这个对象是必须要有的，因为你要在通知中通过它来
调用被通知的方法。通知方法中可以做任何的事情，当要将控制权交给被通知的方法时，它需要调用ProceedingJoinPoint的proceed()方法。  
&emsp;&emsp;需要注意的是，别忘记调用proceed()方法。如果不调这个方法的话，那么你的通知实际上会阻塞对被通知方法的调用。有可能这就是
你想要的效果，但更多的情况是你希望在某个点上执行被通知的方法。  
有意思的是，你可以不调用proceed()方法，从而阻塞对被通知方法的访问，与之类似，你也可以在通知中对它进行多次调用。要这样
做的一个场景就是实现重试逻辑，也就是在被通知方法失败后，进行重复尝试。
#### 3.3 处理通知中的参数
&emsp;&emsp;到目前为止，我们的切面都很简单，没有任何参数。唯一的例外是我们为环绕通知所编写的watchPerformance()示例方法中使用了
ProceedingJoinPoint作为参数。除了环绕通知，我们编写的其他通知不需要关注传递给被通知方法的任意参数。这很正常，因为我
们所通知的perform()方法本身没有任何参数。  
&emsp;&emsp;但是，如果切面所通知的方法确实有参数该怎么办呢？切面能访问和使用传递给被通知方法的参数吗？  
&emsp;&emsp;为了阐述这个问题，让我们重新看一下2.4.4小节中的BlankDisc样例。play()方法会循环所有的磁道并调用playTrack()方法。但
是，我们也可以通过playTrack()方法直接播放某一个磁道中的歌曲。  
假设你想记录每个磁道被播放的次数。一种方法就是修改playTrack()方法，直接在每次调用的时候记录这个数量。但是，记录磁道的播放次数
与播放本身是不同的关注点，因此不应该属于playTrack()方法。看起来，这应该是切面要完成的任务。  
为了记录每个磁道所播放的次数，我们创建了TrackCounter类，它是通知playTrack()方法的一个切面。下面的程序清单展示了这个切面。    
&emsp;&emsp;程序清单4.6　使用参数化的通知来记录磁道播放的次数  
``` java

@Aspect
public class TrackCounter {
    private Map<Integer,Integer> trackCounts=new HashMap<>();

    @Pointcut("execution(* soundsystem.CompactDisc.playTrack(int))"+"&& args(trackNumber)")
    public void trackPlayed(int trackNumber){}

    @Before("trackPlayed(trackNumber)")
    public void countTrack(int trackNumber){
        int currentCount = getPlayCount(trackNumber);
        trackCounts.put(trackNumber,currentCount+1);
    }

    public int getPlayCount(int trackNumber){
        return trackCounts.containsKey(trackNumber)?trackCounts.get(trackNumber):0;
    }
}
```
&emsp;&emsp;像之前所创建的切面一样，这个切面使用@Pointcut注解定义命名的切点，并使用@Before将一个方法声明为前置通知。但是，这里的不同点在于
切点还声明了要提供给通知方法的参数。图4.6将切点表达式进行了分解，以展现参数是在什么地方指定的。    
![](/images/posts/spring/spring-aop/pointcut_expression_param.png)  
&emsp;&emsp;在图4.6中需要关注的是切点表达式中的args(trackNumber)限定符。它表明传递给playTrack()方法的int类型参数也会传递到通知中去。
&emsp;&emsp;参数的名称trackNumber也与切点方法签名中的参数相匹配。    
这个参数会传递到通知方法中，这个通知方法是通过@Before注解和命名切点trackPlayed(trackNumber)定义的。切点定义中的参数与
切点方法中的参数名称是一样的，这样就完成了从命名切点到通知方法的参数转移。  
&emsp;&emsp;到目前为止，在我们所使用的切面中，所包装的都是被通知对象的已有方法。但是，方法包装仅仅是切面所能实现的功能之一。让我们看
一下如何通过编写切面，为被通知的对象引入全新的功能。  
#### 3.4 通过注解引入新功能
&emsp;&emsp;一些编程语言，例如Ruby和Groovy，有开放类的理念。它们可以不用直接修改对象或类的定义就能够为对象或类增加新的方法。不过，
Java并不是动态语言。一旦类编译完成了，我们就很难再为该类添加新的功能了。  
&emsp;&emsp;但是如果仔细想想，我们在本章中不是一直在使用切面这样做吗？当然，我们还没有为对象增加任何新的方法，但是已经为对象拥有的方法
添加了新功能。如果切面能够为现有的方法增加额外的功能，为什么不能为一个对象增加新的方法呢？实际上，利用被称为引入的AOP
概念，切面可以为Spring bean添加新方法。  
&emsp;&emsp;回顾一下，在Spring中，切面只是(实现了它们所包装bean相同接口的)代理。如果除了实现这些接口，代理也能暴露新接口的话，会怎么样
呢？那样的话，切面所通知的bean看起来像是实现了新的接口，即便底层实现类并没有实现这些接口也无所谓。图4.7展示了它们是如何工作的。
![](/images/posts/spring/spring-aop/aop_import_method.png)  
&emsp;&emsp;我们需要注意的是，当引入接口的方法被调用时，代理会委托把此调用给实现了新接口的某个其他对象。实际上，一个bean的实现被拆分
到了多个类中。  
&emsp;&emsp;为了验证该主意能行得通，我们为示例中的所有的Performance实现引入下面的Encoreable接口
``` java
public interface Encoreable {
    void performEncore();
}
```
&emsp;&emsp;暂且先不管Encoreable是不是一个真正存在的单词 ，我们需要有一种方式将这个接口应用到Performance实现中。我们现在假设你
能够访问Performance的所有实现，并对其进行修改，让它们都实现Encoreable接口。但是，从设计的角度来看，这并不是最好的做
法，并不是所有的Performance都是具有Encoreable特性的。另外一方面，有可能无法修改所有的Performance实现，当使用第三
方实现并且没有源码的时候更是如此。值得庆幸的是，借助于AOP的引入功能，我们可以不必在设计上妥协或者侵入性地改变现有的实现。  
&emsp;&emsp;为了实现该功能，我们要创建一个新的切面：
``` java
@Aspect
public class EncoreableIntroducer {

    @DeclareParents(value = "concert.Performance+",defaultImpl = DefaultEncoreable.class)
    public static Encoreable encoreable;
}
```
&emsp;&emsp;可以看到，EncoreableIntroducer是一个切面。但是，它与我们之前所创建的切面不同，它并没有提供前置、后置或环绕通知，而是
通过@DeclareParents注解，将Encoreable接口引入到Performance bean中。  
&emsp;&emsp;@DeclareParents注解由三部分组成：  
1. value属性指定了哪种类型的bean要引入该接口。在本例中，也就是所有实现Performance的类型。（标记符后面的加号表示
是Performance的所有子类型，而不是Performance本身。）
2. defaultImpl属性指定了为引入功能提供实现的类。在这里，我们指定的是DefaultEncoreable提供实现。
3. @DeclareParents注解所标注的静态属性指明了要引入接口。在这里，我们所引入的是Encoreable接口。
和其他的切面一样，我们需要在Spring应用中将EncoreableIntroducer声明为一个bean：  
  
``` xml

<bean class="concert.EncoreableIntroducer" />
```

&emsp;&emsp;Spring的自动代理机制将会获取到它的声明，当Spring发现一个bean使用了@Aspect注解时，Spring就会创建一个代理，然后将调用委托给
被代理的bean或被引入的实现，这取决于调用的方法属于被代理的bean还是属于被引入的接口。  
&emsp;&emsp;在Spring中，注解和自动代理提供了一种很便利的方式来创建切面。它非常简单，并且只涉及到最少的Spring配置。但是，面向注解的切
面声明有一个明显的劣势：你必须能够为通知类添加注解。为了做到这一点，必须要有源码。  
&emsp;&emsp;如果你没有源码的话，或者不想将AspectJ注解放到你的代码之中，Spring为切面提供了另外一种可选方案。让我们看一下如何在Spring
XML配置文件中声明切面。  

## 4 在XML中声明切面

&emsp;&emsp;在本书前面的内容中，我曾经建立过这样一种原则，那就是基于注解的配置要优于基于Java的配置，基于Java的配置要优于基于XML的配
置。但是，如果你需要声明切面，但是又不能为通知类添加注解的时候，那么就必须转向XML配置了。  
&emsp;&emsp;在Spring的aop命名空间中，提供了多个元素用来在XML中声明切面，如表4.3所示。  
![](/images/posts/spring/spring-aop/aop_no_invasive.png)  
&emsp;&emsp;我们已经看过了``` java <aop:aspectj-autoproxy> ```元素，它能够自动代理AspectJ注解的通知类。aop命名空间的其他元素能够让我们直接在
Spring配置中声明切面，而不需要使用注解。  
例如，我们重新看一下Audience类，这一次我们将它所有的AspectJ注解全部移除掉：  
``` java

    public  void  performance(){}

    public void silenceCellPhones(){
        System.out.println("Silencing  cell phone");
    }
    public void takeSeats(){
        System.out.println("Taking seats");
    }

    public void applause(){
        System.out.println("CLAP CLAP CLAP!!!");
    }

    public void demandRefund(){
        System.out.println("Demanding a refund");
    }

}
```
&emsp;&emsp;正如你所看到的，Audience类并没有任何特别之处，它就是有几个方法的简单Java类。我们可以像其他类一样把它注册为Spring应用上下文中
的bean。  
&emsp;&emsp;尽管看起来并没有什么差别，但Audience已经具备了成为AOP通知的所有条件。我们再稍微帮助它一把，它就能够成为预期的通知了。

#### 4.1 声明前置和后置通知

&emsp;&emsp;我们会使用Spring aop命名空间中的一些元素，将没有注解的Audience类转换为切面。下面的程序清单4.9展示了所需要的XML。  
&emsp;&emsp;程序清单4.9　通过XML将无注解的Audience声明为切面  
``` xml

    <aop:config>
        <aop:aspect ref="audience">
            <aop:before 
              pointcut="execution(**concert.Performance.perform(..))" 
              method="silenceCellPhones" />
            <aop:before 
              pointcut="execution(**concert.Performance.perform(..))" 
              method="takeSeats" />
            <aop:after-returning  
              pointcut="execution(**concert.Performance.perform(..))" 
              method="applause" />
            <aop:after-throwing  
              pointcut="execution(**concert.Performance.perform(..))" 
              method="demandRefund" />
        </aop:aspect>
    </aop>
```  

&emsp;&emsp;关于Spring AOP配置元素，第一个需要注意的事项是大多数的AOP配置元素必须在``` java <aop:config> ```元素的上下文内使用。这条规则有几
种例外场景，但是把bean声明为一个切面时，我们总是从``` java <aop:config> ```元素开始配置的。  
&emsp;&emsp;在``` java <aop:config> ```元素内，我们可以声明一个或多个通知器、切面或者切点。在程序清单4.9中，我们使用``` java<aop:aspect> ```元素声明了
一个简单的切面。ref元素引用了一个POJO bean，该bean实现了切面的功能——在这里就是audience。ref元素所引用的bean提供了在
切面中通知所调用的方法。  
&emsp;&emsp;该切面应用了四个不同的通知。两个``` java<aop:before> ```元素定义了匹配切点的方法执行之前调用前置通知方法—也就是Audience bean
的takeSeats()和turnOffCellPhones()方法（由method属性所声明）。``` java<aop:after-returning> ```元素定义了一个返回（after-returning）
通知，在切点所匹配的方法调用之后再调用applaud()方法。同样，``` java<aop:after-throwing> ```元素定义了异常（after-throwing）通知，
如果所匹配的方法执行时抛出任何的异常，都将会调用demandRefund()方法。图4.8展示了通知逻辑如何织入到业务逻辑中。  
![](/images/posts/spring/spring-aop/advice_work_in_pointcut.png)  
&emsp;&emsp;在所有的通知元素中，pointcut属性定义了通知所应用的切点，它的值是使用AspectJ切点表达式语法所定义的切点。  
你或许注意到所有通知元素中的pointcut属性的值都是一样的，这是因为所有的通知都要应用到相同的切点上。  
&emsp;&emsp;在基于AspectJ注解的通知中，当发现这种类型的重复时，我们使用@Pointcut注解消除了这些重复的内容。而在基于XML的切面声
明中，我们需要使用``` java<aop:pointcut> ```元素。如下的XML展现了如何将通用的切点表达式抽取到一个切点声明中，这样这个声明就能在
所有的通知元素中使用了。  
程序清单4.10　使用``` java<aop:pointcut> ```定义命名切点
``` xml

    <aop:config>
        <aop:aspect ref="audience">
            <aop:pointcut 
               id="performace"
               expression="execution(**concert.Performance.perform(..))" />
            <aop:before 
              pointcut-ref="performace" 
              method="silenceCellPhones" />
            <aop:before 
              pointcut-ref="performace" 
              method="takeSeats" />
            <aop:after-returning  
              pointcut-ref="performace" 
              method="applause" />
            <aop:after-throwing  
              pointcut-ref="performace" 
              method="demandRefund" />
        </aop:aspect>
    </aop>
```  
&emsp;&emsp;现在切点是在一个地方定义的，并且被多个通知元素所引用。``` java <aop:pointcut> ```元素定义了一个id为performance的切点。同时修改所有的
通知元素，用pointcut-ref属性来引用这个命名切点。  
&emsp;&emsp;正如程序清单4.10所展示的，``` java <aop:pointcut> ```元素所定义的切点可以被同一个``` java <aop:aspect> ```元素之内的所有通知元素引用。如果想让定义的
切点能够在多个切面使用，我们可以把``` java <aop:pointcut> ```元素放在``` java <aop:config> ```元素的范围内。  

#### 4.2 声明环绕通知

&emsp;&emsp;目前Audience的实现工作得非常棒，但是前置通知和后置通知有一些限制。具体来说，如果不使用成员变量存储信息的话，在前置通知
和后置通知之间共享信息非常麻烦。  
&emsp;&emsp;例如，假设除了进场关闭手机和表演结束后鼓掌，我们还希望观众确保一直关注演出，并报告每个参赛者表演了多长时间。使用前置通知
和后置通知实现该功能的唯一方式是在前置通知中记录开始时间并在某个后置通知中报告表演耗费的时间。但这样的话我们必须在一个成
员变量中保存开始时间。因为Audience是单例的，如果像这样保存状态的话，将会存在线程安全问题。  
&emsp;&emsp;相对于前置通知和后置通知，环绕通知在这点上有明显的优势。使用环绕通知，我们可以完成前置通知和后置通知所实现的相同功能，而
且只需要在一个方法中 实现。因为整个通知逻辑是在一个方法内实现的，所以不需要使用成员变量保存状态。  
&emsp;&emsp;例如，考虑程序清单4.11中新Audience类的watchPerformance()方法，它没有使用任何的注解。  
&emsp;&emsp;程序清单4.11　watchPerformance()方法提供了AOP环绕通知  
``` java
public class Audience {

    public void watchPerformance(ProceedingJoinPoint jp){
        try {
            System.out.println("Silencing  cell phone");
            System.out.println("Taking seats");
            jp.proceed();//执行被通知的方法
            System.out.println("CLAP CLAP CLAP!!!");
        } catch (Throwable throwable) {
            System.out.println("Demanding a refund");
        }
    }
}
```  
&emsp;&emsp;在观众切面中，watchPerformance()方法包含了之前四个通知方法的所有功能。不过，所有的功能都放在了这一个方法中，因此这个
方法还要负责自身的异常处理。声明环绕通知与声明其他类型的通知并没有太大区别。我们所需要做的仅仅是使用``` java <aop:around> ```元素。  
&emsp;&emsp;程序清单4.12　在XML中使用``` java <aop:around> ```元素声明环绕通知  
``` xml

    <aop:config>
        <aop:aspect ref="audience">
            <aop:pointcut 
               id="performace"
               expression="execution(**concert.Performance.perform(..))" />
            <aop:around 
               pointcut-ref="performace"
               method="watchPerformance" />
        </aop:aspect>
    </aop>
```  
&emsp;&emsp;像其他通知的XML元素一样，``` java <aop:around> ```指定了一个切点和一个通知方法的名字。在这里，我们使用跟之前一样的切点，但是为该
切点所设置的method属性值为watchPerformance()方法  

#### 4.3 为通知传递参数
&emsp;&emsp;在4.3.3小节中，我们使用@AspectJ注解创建了一个切面，这个切面能够记录CompactDisc上每个磁道播放的次数。现在，我们使用
XML来配置切面，那就看一下如何完成这一相同的任务。首先，我们要移除掉TrackCounter上所有的@AspectJ注解。  
``` java
public class TrackCounter {
    private Map<Integer,Integer> trackCounts=new HashMap<>();

    public void trackPlayed(int trackNumber){}

    public void countTrack(int trackNumber){
        int currentCount = getPlayCount(trackNumber);
        trackCounts.put(trackNumber,currentCount+1);
    }

    public int getPlayCount(int trackNumber){
        return trackCounts.containsKey(trackNumber)?trackCounts.get(trackNumber):0;
    }
}
```  
&emsp;&emsp;去掉@AspectJ注解后，TrackCounter显得有些单薄了。现在，除非显式调用countTrack()方法，否则TrackCounter不会记录磁
道播放的数量。但是，借助一点Spring XML配置，我们能够让TrackCounter重新变为切面。  
&emsp;&emsp;如下的程序清单展现了完整的Spring配置，在这个配置中声明了TrackCounter bean和BlankDisc bean，并将TrackCounter转化为切面。  
&emsp;&emsp;程序清单4.14　在XML中将TrackCounter配置为参数化的切面  
``` xml

    <aop:config>
        <aop:aspect ref="trackCounter">
            <aop:pointcut 
               id="trackPlayed"
               expression="execution(* soundsystem.CompactDisc.playTrack(int)) and args(trackNumber)" />
            <aop:before 
               pointcut-ref="trackPlayed"
               method="countTrack" />
        </aop:aspect>
    </aop:config>
    </aop>
```  
&emsp;&emsp;可以看到，我们使用了和前面相同的aop命名空间XML元素。唯一明显的差别在于切点表达式中包含了一个参数，这个参数会传递到通知方法中。
如果你将这个表达式与程序清单4.6中的表达式进行对比会发现它们几乎是相同的。唯一的差别在于这里使用and关键字而不是“&&”
（因为在XML中，“&”符号会被解析为实体的开始）。我们通过练习已经使用Spring的aop命名空间声明了几个基本的切面，
那么现在让我们看一下如何使用aop命名空间声明引入切面。  
#### 4.4 通过切面引入新的功能
&emsp;&emsp;在前面的4.3.4小节中，我向你展现了如何借助AspectJ的@DeclareParents注解为被通知的方法神奇地引入新的方法。但是
AOP引入并不是AspectJ特有的。使用Spring aop命名空间中的``` java <aop:declare-parents> ```，我们可以实现相同的功能。  
&emsp;&emsp;如下的XML代码片段与之前基于AspectJ的引入功能是相同：  
``` xml

        <aop:aspect>
            <aop:declare-parents
               type-matching="concert.Performance+" 
               implement-interface="concert.Encoreable"
               default-impl="concert.DefaultEncoreable"/>
        </aop:aspect>
```  
顾名思义，``` java <aop:declare-parents> ```声明了此切面所通知的bean要在它的对象层次结构中拥有新的父类型。具体到本例中，类型匹
&emsp;&emsp;配Performance接口（由types-matching属性指定）的那些bean在父类结构中会增加Encoreable接口（由implement-interface属性指定）。
最后要解决的问题是Encoreable接口中的方法实现要来自于何处。  
&emsp;&emsp;这里有两种方式标识所引入接口的实现。在本例中，我们使用default-impl属性用全限定类名来显式指定Encoreable的实
现。或者，我们还可以使用delegate-ref属性来标识。   
``` xml

        <aop:aspect>
            <aop:declare-parents
               type-matching="concert.Performance+" 
               implement-interface="concert.Encoreable"
               delegate-ref="encoreableDelegate"/>
        </aop:aspect>
```

&emsp;&emsp;delegate-ref属性引用了一个Spring bean作为引入的委托。这需要在Spring上下文中存在一个ID为encoreableDelegate的bean。  

``` xml
<bean id="encoreableDelegate" class="concert.DefaultEncoreable" />
```
&emsp;&emsp;使用default-impl来直接标识委托和间接使用delegate-ref的区别在于后者是Spring bean，它本身可以被注入、通知或使用其他的Spring配置。
## 5 注入AspectJ切面
&emsp;&emsp;虽然Spring AOP能够满足许多应用的切面需求，但是与AspectJ相比，Spring AOP 是一个功能比较弱的AOP解决方案。AspectJ提供了Spring
AOP所不能支持的许多类型的切点。  
&emsp;&emsp;例如，当我们需要在创建对象时应用通知，构造器切点就非常方便。不像某些其他面向对象语言中的构造器，Java构造器不同于其他的正
常方法。这使得Spring基于代理的AOP无法把通知应用于对象的创建过程。  
&emsp;&emsp;对于大部分功能来讲，AspectJ切面与Spring是相互独立的。虽然它们可以织入到任意的Java应用中，这也包括了Spring应用，但是在应用
AspectJ切面时几乎不会涉及到Spring。  
&emsp;&emsp;但是精心设计且有意义的切面很可能依赖其他类来完成它们的工作。如果在执行通知时，切面依赖于一个或多个类，我们可以在切面内部
实例化这些协作的对象。但更好的方式是，我们可以借助Spring的依赖注入把bean装配进AspectJ切面中。  
&emsp;&emsp;为了演示，我们为上面的演出创建一个新切面。具体来讲，我们以切面的方式创建一个评论员的角色，他会观看演出并且会在演出之后提
供一些批评意见。下面的CriticAspect就是一个这样的切面。  
&emsp;&emsp;程序清单4.15　使用AspectJ实现表演的评论员  
``` java
public aspect CriticAspect {
    public CriticAspect(){}

    pointcut performance() : execution(* perform(..));

    afterReturning() : performance(){
        System.out.println(criticismEngine.getCriticism());
    }
    private CriticismEngine criticismEngine;

    public void setCriticismEngine(CriticismEngine criticismEngine) {
        this.criticismEngine = criticismEngine;
    }
}
```  
&emsp;&emsp;CriticAspect的主要职责是在表演结束后为表演发表评论。程序清单4.15中的performance()切点匹配perform()方法。当它
与afterReturning()通知一起配合使用时，我们可以让该切面在表演结束时起作用。  
&emsp;&emsp;程序清单4.15有趣的地方在于并不是评论员自己发表评论，实际上，CriticAspect与一个CriticismEngine对象相协作，在表
演结束时，调用该对象的getCriticism()方法来发表一个苛刻的评论。为了避免CriticAspect和CriticismEngine之间产生不必要的耦合，
我们通过Setter依赖注入为CriticAspect设置CriticismEngine。图4.9展示了此关系。
![](/images/posts/spring/spring-aop/Aspectj_inject.png)  
&emsp;&emsp;CriticismEngine自身是声明了一个简单getCriticism()方法的接口。程序清单4.16为CriticismEngine的实现。  
&emsp;&emsp;程序清单4.16　要注入到CriticAspect中的CriticismEngine实现  
``` java
public class CriticismEngineImpl implements CriticismEngine{
    public CriticismEngineImpl(){}
    public String getCriticism(){
        int i=(int)(Math.random() * criticismPool.length);
        return criticismPool[i];
    }
    private String[] criticismPool;

    public void setCriticismPool(String[] criticismPool) {
        this.criticismPool = criticismPool;
    }
}
```
&emsp;&emsp;CriticismEngineImpl实现了CriticismEngine接口，通过从注入的评论池中随机选择一个苛刻的评论。这个类可以使用如下的
XML声明为一个Spring bean。  
``` xml
<bean id="criticismEngine"
      class="com...">
      <property name="criticismPool">
        <list>
            <value>worst</value>
            <value>laughed</value>
            <value>a must see show</value>
        </list>
      </property>
</bean>
```
&emsp;&emsp;到目前为止，一切顺利。我们现在有了一个要赋予CriticAspect的Criticism-Engine实现。剩下的就是为CriticAspect装配CriticismEngineImple。  
在展示如何实现注入之前，我们必须清楚AspectJ切面根本不需要Spring就可以织入到我们的应用中。如果想使用Spring的依赖注入为
AspectJ切面注入协作者，那我们就需要在Spring配置中把切面声明为一个Spring配置中的``` java <bean> ```。如下的<bean>声明会把
criticismEnginebean注入到CriticAspect中：  
``` xml
<bean class="com..."
      factory-method="aspectOf">
      <property name="criticismEngine" ref="criticismEngine" />
</bean>
```
&emsp;&emsp;很大程度上，``` java<bean> ``` 的声明与我们在Spring中所看到的其他<bean>配置并没有太多的区别，但是最大的不同在于使用了factory-method属性。
通常情况下，Spring bean由Spring容器初始化，但是AspectJ切面是由AspectJ在运行期创建的。等到Spring有机会为CriticAspect
注入CriticismEngine时，CriticAspect已经被实例化了。  
&emsp;&emsp;因为Spring不能负责创建CriticAspect，那就不能在 Spring中简单地把CriticAspect声明为一个bean。相反，我们需要一种方式为
Spring获得已经由AspectJ创建的CriticAspect实例的句柄，从而可以注入CriticismEngine。幸好，所有的AspectJ切面都提供了一
个静态的aspectOf()方法，<u>该方法返回切面的一个单例</u>。所以为了获得切面的实例，我们必须使用factory-method来调
&emsp;&emsp;用asepctOf()方法而不是调用CriticAspect的构造器方法。  
&emsp;&emsp;简而言之，Spring不能像之前那样使用``` java <bean> ```声明来创建一个CriticAspect实例——它已经在运行时由AspectJ创建完成了。
Spring需要通过aspectOf()工厂方法获得切面的引用，然后像<bean>元素规定的那样在该对象上执行依赖注入。