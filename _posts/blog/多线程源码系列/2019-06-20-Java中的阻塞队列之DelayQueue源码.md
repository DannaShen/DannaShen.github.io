---
layout: post
title: Java中的阻塞队列之DelayQueue源码
categories: 多线程源码系列
description: 类加载器相关信息、双亲委派模型及如何破坏
keywords: 类加载器、双亲委派模型、破坏双亲委派模型
---
## 1. DelayQueue介绍
DelayQueue 是一个支持延时获取元素的无界阻塞队列，队列使用PriorityQueue来实现，队列中的元素必须实现Delayed接口，
在创建元素时可以指定多久才能从队列中获取当前元素，只有在延时期满时才能从队列中提取元素。  
<br/>
PriorityQueue 是一种优先级的队列，队列中的元素会按照优先级进行排序，PriorityQueue其实现原理也是使用的二叉堆，
因此这次将不会再分析PriorityQueue的原理了，可以参考Java 并发 — 阻塞队列之PriorityBlockingQueue 中关于对二叉堆的分析。
