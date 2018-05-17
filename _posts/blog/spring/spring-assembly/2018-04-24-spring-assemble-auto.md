---
layout: post
title: spring之自动装配bean
categories: spring-assembly
description: spring之自动装配bean
keywords: spring,自动装配
---

> Spring从两个角度来实现自动化装配：
>1. 组件扫描：Spring会自动发现应用上下文中所创建的bean。
>2. 自动装配：Spring自动满足bean之间的依赖。

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 1.自动装配bean的过程:
1. 把需要被扫描的类，添加 @Component注解，使它能够被Spring自动发现。
2. 通过显示的设置Java代码 @ComponentScan注解或XML配置，让Spring开启组件扫描，并将扫描的结果类创建bean。
3. @Autowried注解能实现bean的自动装配，实现依赖注入
&nbsp;
&nbsp;
&nbsp;
&nbsp;
	
> 案例：音响系统的组件。  
>首先为CD创建CompactDisc接口及实现类，Spring会发现它并将其创建为一个bean。然后，会创建一个CDPlayer类，让Spring发现它，并将CompactDisc bean注入进来。

&nbsp;
&nbsp;
&nbsp;
&nbsp;


## 2.组件扫描

#### 2.1创建CompactDisc接口：

``` java

	public interface CompactDisc {
		  void cdInfo();
	}
```

#### 2.2实现CompactDisc接口：
``` java

	@Component
	public class SgtPeppers implements CompactDisc {
	
	    private String title="K歌之王";
	    private String artist="陈奕迅";
	    @Override
	    public void cdInfo() {
	        System.out.print("This CD's title:"+title+";artist:"+artist);
	    }
	 
	}
```	
&emsp;&emsp;在SgtPeppers类上使用了 <u>@Component注解，这个注解表明该类会作为组件类</u>，并告知Spring要为这个类创建bean，不需要显示配置SgtPeppers bean。
不过组件扫描默认是不开启的。我们需要显示配置一下Spring，从而命令Spring去寻找带有 @Component注解的类，并创建bean。

#### 2.3定义多媒体播放器接口 

``` java

	public interface MediaPlayer {
	    void play();
	}
```	

#### 2.4启用组件扫描包括Java和XML两种方式

*1. 通过Java启用组件扫描*
``` java

	@Configuration
	@ComponentScan
	public class CDPlayerConfig {
	}
```	
&emsp;&emsp;注意，类CDPlayerConfig通过Java代码定义了Spring的装配规则，但是可以看出并没有显示地声明任何bean，只不过它使用了<u>@ComponentScan注解，这个注解能够在Spring中启用组件扫描。（@Configuration注解表明这个类是一个配置类）</u>如果没有其他配置的话，@ComponentScan默认会扫描与配置类相同的包以及这个包下的所有子包，查找所有带有 @Component注解的类。这样的话，SgtPeppers类就会被自动创建一个bean。
Spring之所以存在是因为解耦和，即不用传统方法来new一个新的实例，因此在实现类中使用@Component标明，即可达到效果

*2. 通过XML启用组件扫描*
``` java

	<context:component-scan base-package="..." />
```	
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 3.自动装配

&emsp;&emsp;自动装配就是让Spring自动满足bean依赖的一种方式，在满足依赖的过程中，会在Spring的上下文中寻找匹配一个bean需求的其他bean。为了声明要进行自动装配，我们可以借助Spring的 **@Autowried注解**。

#### 3.1@Autowried注解用在构造器上
下面是CDPlayer类，它在构造器上添加了 @Autowried注解，这表明当创建CDPlayer bean的时候，会通过这个构造器来进行实例化并且会传入一个可设置给CompactDisc类型的bean。
``` java

	@Component
	public class CDPlayer implements MediaPlayer {
	  private CompactDisc cd;
	
	  @Autowired
	  public CDPlayer(CompactDisc cd) {
	    this.cd = cd;
	  }
	
	  public void play() {
	    cd.play();
	  }
	
	}
```	
	
#### 3.2@Autowried注解用在Setter方法上
``` java

	@Component
	public class CDPlayer implements MediaPlayer {
	  private CompactDisc cd;
	
	  @Autowired
	  public void setCompactDisc(CompactDisc cd) {
	    this.cd = cd;
	  }
	
	  public void play() {
	    cd.play();
	  }
	
	}
```	
&emsp;&emsp;事实上，@Autowried注解可以用在类的任何方法上去引入依赖的bean，Spring都会尝试满足方法参数上所声明的依赖。
如果没有匹配的bean，那么在应用上下文创建的时候，Spring会抛出一个异常。为了避免出现异常，可以将 @Autowried的required属性设置为false：
``` java

	@Autowired(required=false)
	public CDPlayer(CompactDisc cd) {
	    this.cd = cd;
	}
```		
&emsp;&emsp;设置以后，会尝试自动装配，但是如果没有匹配的bean，Spring默认会处于未装配的状态。但是把required设置为false时，需要谨慎对待，如果代码中没有进行null检查的话，建议不使用，不然就会出现NullPointerException异常。

&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 4.测试


``` java

	public class CDPlayerTest {
	    
	    @Rule
	    public final StandardOutputStreamLog log=new StandardOutputStreamLog();
	    
	    @Autowired
	    private MediaPlayer player;
	    
	    @Autowired
	    private CompactDisc cd;
	    
	    @Test
	    public void cdShouldNotBeNull() {
	    	//如果测试正常，说名cd已被初始化，Spring依赖注入有效发挥作用了（该方法不是自己写的，应该是个工具类似的）
	        assertNotNull(cd);
	    }
	    
	    @Test
	    public void play(){
	        player.play();
	        //该方法不是自己写的，应该是个工具类似的
	        assertEquals("Info of CD playing :This CD's title:K歌之王;artist:陈奕迅", log.getLog());
	    }
	}	
```
&emsp;&emsp;CDPlayer的构造器中添加了@Autowired注解，Spring将把一个可分配给CompactDisc类型的bean自动注入进来。除了注入CompactDisc，
我们还将CDPlayerbean注入到测试代码的player成员变量之中（它是更为通用的MediaPlayer类型）。在play()测试方法中，我们可以调用CDPlayer的play()方法，并断言它的行为与你的预期一致。
	
	
	
	
	
	
	
	