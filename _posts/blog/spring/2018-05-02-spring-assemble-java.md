---
layout: post
title: spring之java代码装配bean
categories: spring
description: spring之java代码装配bean
keywords: spring,java代码装配
---

> 尽管在很多场景下通过组件扫描和自动装配实现Spring的自动化配置是更为推荐的方式，但有时候自动化配置的方案行不通，因此需要明确配置Spring。比如说，你想要将第三方库中的组件装配到你的应用中，在这种情况下，是没有办法在它的类上添加@Component和@Autowired注解的，因此就不能使用自动化装配的方案了。  
>在这种情况下，你必须要采用显式装配的方式。在进行显式配置的时候，有两种可选方案：Java和XML。在这节中，我们将会学习如何使用Java配置。  
>在进行显式配置时，JavaConfig是更好的方案，因为它更为强大、类型安全并且对重构友好。因为它就是Java代码，就像应用程序中的其他Java代码一样。   
>JavaConfig与其他的Java代码又有所区别，在概念上，它与应用程序中的业务逻辑和领域代码是不同的。尽管它与其他的组件一样都使用相同的语言进行表述，但JavaConfig是配置代码。这意味着它不应该包含任何业务逻辑，JavaConfig也不应该侵入到业务逻辑代码之中。尽管不是必须的，但通常会将JavaConfig放到单独的包中，使它与其他的应用程序逻辑分离开来，这样对于它的意图就不会产生困惑了。

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 1.创建配置类

&nbsp;

``` java

	@Configuration
	public class CDPlayerConfig {
	}
```
&nbsp;

创建JavaConfig类的关键在于为其添加@Configuration注解，@Configuration注解表明这个类是一个配置类，该类应该包含在Spring应用上下文中如何创建bean的细节。到此为止，我们都是依赖组件扫描来发现Spring应该创建的bean。<u>尽管我们可以同时使用组件扫描和显式配置</u>，但是在本节中，我们更加关注于显式配置，因此我将CDPlayerConfig的<u>@ComponentScan注解移除掉了</u>。移除了@ComponentScan注解，此时的CDPlayerConfig类就没有任何作用了。如果你现在运行CDPlayerTest的话，测试会失败，并且会出现BeanCreation- Exception异常。测试期望被注入CDPlayer和CompactDisc，但是这些bean根本就没有创建，因为组件扫描不会发现它们。

## 2.基础接口及类

#### 2.1创建CompactDisc接口：

``` java

	public interface CompactDisc {
		  void play();
	}
```

#### 2.2实现CompactDisc接口：

``` java

	public class SgtPeppers implements CompactDisc {
	
	  private String title = "Sgt. Pepper's Lonely Hearts Club Band";  
	  private String artist = "The Beatles";
	  
	  public void play() {
	    System.out.println("Playing " + title + " by " + artist);
	  }
	 
	}
```	

#### 2.3定义多媒体播放器接口 

``` java

	public interface MediaPlayer {
	    void play();
	}
```	

#### 2.4定义多媒体播放器实现类
``` java

	public class CDPlayer implements MediaPlayer {
	  private CompactDisc cd;
	
	  public CDPlayer(CompactDisc cd) {
	    this.cd = cd;
	  }
	
	  public void play() {
	    cd.play();
	  }
	
	}
```	

## 3.声明简单的bean

&emsp;&emsp;要在JavaConfig中声明bean，我们需要编写一个方法，这个方法会创建所需类型的实例，然后给这个方法添加@Bean注解。比方说，下面的代码声明了CompactDisc bean：
``` java

	@Configuration
	public class CDPlayerConfig {
		@Bean
		public CompactDisc sgtPeppers(){
			return new SgtPeppers();
		}
	}
```
&emsp;&emsp;<u>@Bean注解会告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文中的bean。</u>方法体中包含了最终产生bean实例的逻辑。  默认情况下，bean的ID与带有@Bean注解的方法名是一样的。在本例中，bean的名字将会是sgtPeppers。如果你想为其设置成一个不同的名字的话，那么可以重命名该方法，也可以通过name属性指定一个不同的名字：

``` java

	@Bean(name="lonelyHeartsClubBand")
	public CompactDisc sgtPeppers(){
		return new SgtPeppers();
	}
```
&emsp;&emsp;不管你采用什么方法来为bean命名，bean声明都是非常简单的。方法体返回了一个新的SgtPeppers实例。这里是使用Java来进行描述的，因此我们可以发挥Java提供的所有功能，只要最终生成一个CompactDisc实例即可。  

## 4.借助JavaConfig实现注入
&emsp;&emsp;我们前面所声明的CompactDisc bean是非常简单的，它自身没有其他的依赖。但现在，我们需要声明CDPlayer bean，它依赖于CompactDisc。在JavaConfig中，要如何将它们装配在一起呢？  
&emsp;&emsp;在JavaConfig中装配bean的最简单方式就是引用创建bean的方法。例如，下面就是一种声明CDPlayer的可行方案：
``` java

	@Configuration
	public class CDPlayerConfig {
		@Bean
		public CompactDisc sgtPeppers(){
			return new SgtPeppers();
		}
		@Bean
		public CDPlayer cdPlayer(){
			return new CDPlayer(sgtPeppers());
		}
	}
```

&nbsp;

&emsp;&emsp;cdPlayer()方法像sgtPeppers()方法一样，同样使用了@Bean注解，这表明这个方法会创建一个bean实例并将其注册到Spring应用上下文中。所创建的bean ID为cdPlayer，与方法的名字相同。cdPlayer()的方法体与sgtPeppers()稍微有些区别。在这里并没有使用默认的构造器构建实例，而是调用了需要传入CompactDisc对象的构造器来创建CDPlayer实例。看起来，CompactDisc是通过调用sgtPeppers()得到的，但情况
并非完全如此。因为sgtPeppers()方法上添加了@Bean注解，Spring将会拦截所有对它的调用，并确保直接返回该方法所创建的bean，而不是每次都对其进行实际的调用。比如说，假设你引入了一个其他的CDPlayerbean，它和之前的那个bean完全一样：
``` java

	@Configuration
	public class CDPlayerConfig {
		@Bean
		public CompactDisc sgtPeppers(){
			return new SgtPeppers();
		}
		@Bean
		public CDPlayer cdPlayer(){
			return new CDPlayer(sgtPeppers());
		}
		@Bean
		public CDPlayer anotherCdPlayer(){
			return new CDPlayer(sgtPeppers());
		}
	}
```

&nbsp;

&emsp;&emsp;假如对sgtPeppers()的调用就像其他的Java方法调用一样的话，那么每个CDPlayer实例都会有一个自己特有的SgtPeppers实例。如果我们讨论的是实际的CD播放器和CD光盘的话，这么做是有意义的。如果你有两台CD播放器，在物理上并没有办法将同一张CD光盘放到两个CD播放器中。  
&emsp;&emsp;但是，在软件领域中，我们完全可以将同一个SgtPeppers实例注入到任意数量的其他bean之中。默认情况下，Spring中的bean都是单例的，我们并没有必要为第二个CDPlayer bean创建完全相同的SgtPeppers实例。所以，Spring会拦截对sgtPeppers()的调用并确保返回的是Spring所创建的bean，也就是Spring本身在调用sgtPeppers()时所创建的CompactDiscbean。因此，两个CDPlayer bean会得到相同的SgtPeppers实例。  
&emsp;&emsp;可以看到，通过调用方法来引用bean的方式有点令人困惑。其实还有一种理解起来更为简单的方式：
``` java

	@Configuration
	public class CDPlayerConfig {
		@Bean
		public CDPlayer cdPlayer(CompactDisc compactDisc){
			return new CDPlayer(compactDisc);
		}
	}
```
&emsp;&emsp;在这里，cdPlayer()方法请求一个CompactDisc作为参数。当Spring调用cdPlayer()创建CDPlayerbean的时候，它会自动装配一个CompactDisc到配置方法之中。然后，方法体就可以按照合适的方式来使用它。借助这种技术，cdPlayer()方法也能够将CompactDisc注入到CDPlayer的构造器中，而且不用明确引用CompactDisc的@Bean方法。  
emsp;&emsp;通过这种方式引用其他的bean通常是最佳的选择，因为它不会要求将CompactDisc声明到同一个配置类之中。在这里甚至没有要CompactDisc必须要在JavaConfig中声明，实际上它可以通过组件扫描功能自动发现或者通过XML来进行配置。你可以将配置分散到多个配置类、XML文件以及自动扫描和装配bean之中，只要功能完整健全即可。不管CompactDisc是采用什么方式创建出来的，Spring都会将其传入到配置方法中，并用来创建CDPlayerbean  
emsp;&emsp;另外，需要提醒的是，我们在这里使用CDPlayer的构造器实现了DI功能，但是我们完全可以采用其他风格的DI配置。比如说，如果你想通过Setter方法注入CompactDisc的话，那么代码看起来应该是这样的：
``` java

	@Configuration
	public class CDPlayerConfig {
		@Bean
		public CDPlayer cdPlayer(CompactDisc compactDisc){
			CDPlayer cdPlayer=new CDPlayer();
			cdPlayer.setCompactDisc(compactDisc);
			return cdPlayer;
		}
	}
```
emsp;&emsp;再次强调一遍，带有@Bean注解的方法可以采用任何必要的Java功能来产生bean实例。构造器和Setter方法只是@Bean方法的两个简单样例。这里所存在的可能性仅仅受到Java语言的限制。
