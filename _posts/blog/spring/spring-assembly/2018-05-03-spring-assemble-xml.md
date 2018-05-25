---
layout: post
title: spring之XML装配bean
categories: spring
description: spring之XML装配bean
keywords: spring,XML装配
---

> &emsp;&emsp;在Spring刚刚出现的时候，XML是描述配置的主要方式。

>&emsp;&emsp;Spring现在有了强大的自动化配置和基于Java的配置，鉴于已经存在那么多基于XML的Spring配置，所以理解如何在Spring中使用XML还是很重要的。

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## 1.创建XML配置规范
&emsp;&emsp;在使用JavaConfig的时候，这意味着要创建一个带有@Configuration注解的类，而在XML配置中，这意味着要创建一个XML文件，并且要以<beans>元素为根。
``` xml

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans";
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance";
	 xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context">
	 
	</beans>
```

## 2.声明一个简单的<bean>

#### 2.1声明CompactDisc bean：

``` xml

	<bean class="soundsystem.SgtPeppers"/>
```
&emsp;&emsp;要在基于XML的Spring配置中声明一个bean，我们要使用spring-beans模式中的另外一个元素：<bean>。<bean>元素类似于JavaConfig中的@Bean注解。  
这里声明了一个很简单的bean，创建这个bean的类通过class属性来指定的，并且要使用全限定的类名。因为没有明确给定ID，所以这个bean将会根据全限定类名来进行命名。在本例中，bean的ID将会是“soundsystem.SgtPeppers#0”。其中，“#0”是一个计数的形式，用来区分相同类型的其他bean。如果你声明了另外一个SgtPeppers，并且没有明确进行标识，那么它自动得到的ID将会
是“soundsystem.SgtPeppers#1”。尽管自动化的bean命名方式非常方便，但如果你要稍后引用它的话，那自动产生的名字就没有多大的用处了。因此，通常来讲更好的办法是借助id属性，为每个bean设置一个你自己选择的名字
``` xml

	<bean id="compactDisc" class="soundsystem.SgtPeppers"/>
```

## 3.借助构造器注入初始化bean

&emsp;&emsp;在XML中声明DI时，会有多种可选的配置方案和风格。具体到构造器注入，有两种基本的配置方案可供选择：  
- <constructor-arg>元素
- 使用Spring 3.0所引入的c-命名空间
两者的区别在很大程度就是是否冗长烦琐。可以看到，<constructor-arg>元素比使用c-命名空间会更加冗长，从而导致XML更加难以读懂。另外，有些事情<constructor-arg>可以做到，但是使用c-命名空间却无法实现。

#### 3.1构造器注入bean引用

&emsp;&emsp;CDPlayerbean有一个接受CompactDisc类型的构造器：
``` xml

	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		<constructor-arg ref="compactDisc" />
	</bean>
```
&emsp;&emsp;当Spring遇到这个<bean>元素时，它会创建一个CDPlayer实例。<constructor-arg>元素会告知Spring要将一个ID为compactDisc的bean引用传递到CDPlayer的构造器中

#### 3.2spring的c-命名空间 
&emsp;&emsp;要使用它的话，必须要在XML的顶部声明其模式（加上：xmlns:c="http://www.springframework.org/schema/c";）：
``` xml

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans";
	 xmlns:c="http://www.springframework.org/schema/c";
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance";
	 xmlns:context="http://www.springframework.org/schema/context";
	 xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context">
	 
	</beans>
```
 构造器参数声明如下：
 ``` xml

	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		c:cd-ref="compactDisc" />
```
&emsp;&emsp;属性名以“c:”开头，也就是命名空间的前缀。接下来就是要装配的构造器参数的名称，在此之后是“-ref”，这是一个命名的约定，它会告诉
Spring，正在装配的是一个bean的引用，这个bean的名字是compactDisc,而不是字面量“compactDisc”。  
&emsp;&emsp;但是它直接引用了构造器参数的名称，替代的方案是我们使用参数在整个参数列表中的位置信息：
``` xml

	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		c：_0-ref="compactDisc" />
```
&emsp;&emsp;将参数的名称替换成了“0”，也就是参数的索引。因为在XML中不允许数字作为属性的第一个字符，因此必须要添加一个下画线作为前缀  
&emsp;&emsp;如果有多个构造器参数的话，这当然是很有用处的。在这里因为只有一个构造器参数，所以我们还有另外一个方案——根本不用去标示参数：
``` xml

	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		c：_-ref="compactDisc" />
```

###### 3.2.1将字面量注入到构造器中
&emsp;&emsp; 我们所做的DI通常指的都是类型的装配——也就是将对象的引用装配到依赖于它们的其他对象之中——而有时候，我们需要做的只是用一个字面量值来配置对象。为了阐述这一点，假设你要创建CompactDisc的一个新实现，如下所示：
 ``` java

	public class BlankDisc implements CompactDisc {

	private String title;	
	private String artist;
	public BlankDisc(String title,String artist){
		this.title=title;
		this.artist=artist;
	}
	
	@Override
	public void play() {
		System.out.println("Palying "+title+" by "+artist);
	}

	}

```
 &emsp;&emsp;在SgtPeppers中，唱片名称和艺术家的名字都是硬编码的，但是这个CompactDisc实现与之不同，它更加灵活。现在，我们可以将已有的SgtPeppers替换为这个类：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
		<constructor-arg value="The Beatles" />
	</bean>
```
&emsp;&emsp;我们再次使用<constructor-arg>元素进行构造器参数的注入。但是这一次我们没有使用“ref”属性来引用其他的bean，而是使用了value属性，通过该属性表明给定的值要以字面量的形式注入到构造器之中。 
如果要使用c-命名空间的话，这个例子又该是什么样子呢？第一种方案是引用构造器参数的名字：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<c:title="Sgt. Pepper's Lonely Hearts Club Band" />
		<c:artist="The Beatles" />
	</bean>
```
&emsp;&emsp;可以看到，装配字面量与装配引用的区别在于属性名中去掉了“-ref”后缀。与之类似，我们也可以通过参数索引装配相同的字面量值，如下所示：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<c:_0="Sgt. Pepper's Lonely Hearts Club Band" />
		<c:_1="The Beatles" />
	</bean>
```
&emsp;&emsp;如果只有一个构造器参数的话,我们可以在Spring中这样声明它：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<c:_="Sgt. Pepper's Lonely Hearts Club Band" />
	</bean>
```
 &emsp;&emsp;在装配bean引用和字面量值方面，<constructor-arg>和c-命名空间的功能是相同的。但是有一种情况是<constructor-arg>能够实现，c-命名空间却无法做到的。接下来，让我们看一下如何将集合装配到构造器参数中
 
###### 3.2.2装配集合
 请考虑下面这个新的BlankDisc：
  ``` java

	public class BlankDisc implements CompactDisc {

	private String title;
	private String artist;
	private List<String> tracks;
	public BlankDisc(String title,String artist,List<String> tracks){
		this.title=title;
		this.artist=artist;
		this.tracks=tracks;
	}
	
	@Override
	public void play() {
		System.out.println("Palying "+title+" by "+artist);
		for (String track : tracks) {
			System.out.println("-Track："+track);
		}
	}

}

```
采用如下的方式传递null给它：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
		<constructor-arg value="The Beatles" />
		<constructor-arg><null/></constructor-arg>
	</bean>
``` 
采用<list>方式提供一个列表(也可以<set>)：
 ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
		<constructor-arg value="The Beatles" />
		<constructor-arg>
			<list>
				<value>111111</value>
				<value>222222</value>
				<value>333333</value>
				<value>444444</value>
				<value>555555</value>
			</list>	
		</constructor-arg>
	</bean>
``` 
 &emsp;&emsp;其中，<list>元素是<constructor-arg>的子元素，这表明一个包含值的列表将会传递到构造器中。其中，<value>元素用来指定列表中的每个元素。  
 与之类似，我们也可以使用<ref>元素替代<value>，实现bean引用列表的装配。如下所示：
  ``` xml

	<bean id="compactDisc" class="soundsystem.BlankDisc">
		<constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
		<constructor-arg value="The Beatles" />
		<constructor-arg>
			<list>
				<ref bean=""/>
				<ref bean=""/>
				<ref bean=""/>
				<ref bean=""/>
			</list>	
		</constructor-arg>
	</bean>
``` 

## 4.设置属性
&emsp;&emsp;选择构造器注入还是属性注入,作为一个通用的规则，我倾向于对强依赖使用构造器注入，而对可选性的依赖使用属性注入。  

#### 4.1 Spring XML实现引用属性注入:
  ``` xml
  
	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		<property name="compactDisc" ref="compactDisc"/>
	</bean>
```
  &emsp;&emsp;property标签为属性的Setter方法所提供的功能与<constructor-arg>元素为构造器所提供的功能是一样的。  
 &emsp;&emsp;Spring为<constructor-arg>元素提供了c-命名空间作为替代方案，与之类似，Spring提供了更加简洁的p-命名空间，作为<property>元素的替代方案。为了启用p-命名空间，必须要在XML文件中与其他的命名空间一起对其进行声明：
 
	``` xml

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans";
	 xmlns:p="www.springframework.org/schema/p";
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance";
	 xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context">
	 
	</beans>
``` 
 我们可以使用p-命名空间，按照以下的方式装配compactDisc属性： 
 ``` xml

	<bean id="cdPlayer" class="soundsystem.CDPlayer">
		p：compactDisc-ref="compactDisc" />
```
 首先，属性的名字使用了“p:”前缀，表明我们所设置的是一个属性。接下来就是要注入的属性名。最后，属性的名称以“-ref”结尾，这会提示Spring要进行装配的是引用，而不是字面量。
 
#### 4.2 Spring XML实现字面量属性注入:
 BlankDisc这次完全通过属性注入进行配置，BlankDisc类如下所示：
 ``` java
  	
	public class BlankDisc implements CompactDisc {
	private String title;
	private String artist;
	private List<String> tracks;

	public void setTitle(String title) {
		this.title = title;
	}

	public void setArtist(String artist) {
		this.artist = artist;
	}

	public void setTracks(List<String> tracks) {
		this.tracks = tracks;
	}


	@Override
	public void play() {
		System.out.println("Palying "+title+" by "+artist);
		for (String track : tracks) {
			System.out.println("-Track："+track);
		}
	}

	}
```
 可以借助<property>元素的value属性实现该功能，与之前通过<constructor-arg>装配tracks是完全一样的。  
 另外一种可选方案就是使用p-命名空间的属性来完成该功能，与c-命名空间一样  
 ``` xml

	<bean 	id="compactDisc" 
			class="soundsystem.BlankDisc"
			p:title="Sgt.Pepper's Lonely Hearts Club Band"
			p:artist="The Beatles">
		<property name="tracks">
			<list>
				<value>..</value>
				<value>..</value>
				<value>..</value>
			</list>
		</property>
	</bean>
``` 
&emsp;&emsp; 与c-命名空间一样，装配bean引用与装配字面量的唯一区别在于是否带有“-ref”后缀。如果没有“-ref”后缀的话，所装配的就是字面量。
但需要注意的是，我们不能使用p-命名空间来装配集合，没有便利的方式使用p-命名空间来指定一个值（或bean引用）的列表。但是，我们可以使用Spring util-命名空间中的一些功能来简化BlankDisc bean。  
首先，需要在XML中声明util-命名空间及其模式：
``` xml

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans";
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance";
	 xmlns:p="www.springframework.org/schema/p";
	 xmlns:util="www.springframework.org/schema/util";
	 xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/util
	http://www.springframework.org/schema/util/spring-util.xsd">
	 ...
	</beans>
``` 
&emsp;&emsp;util-命名空间所提供的功能之一就是<util:list>元素，它会创建一个列表的bean。借助<util:list>，我们可以将磁道列表转移到BlankDisc bean之外，并将其声明到单独的bean之中，如下所示：
``` xml
	
	<util:list id="trackList">
		<value>...</value>
		<value>...</value>
		<value>...</value>
	</util:list>
```
现在，我们能够像使用其他的bean那样，将列表bean注入到BlankDisc bean的tracks属性中
 ``` xml

	<bean 	id="compactDisc" 
			class="soundsystem.BlankDisc" 
			p:title="Sgt.Pepper's Lonely Hearts Club Band"
			p:artist="The Beatles"
			p:tracks-ref="trackList">
	</bean>
``` 
<util:list>只是util-命名空间中的多个元素之一。下面的表列出了util-命名空间提供的所有元素。




 
 
 
 
 
 
 
 
 
 
 
 
 
 