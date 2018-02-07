---
layout: post
title: HBase系统整理
categories: bigData
description: HBase系统整理
keywords: HBase
---

## Hbase架构

在HBase中，表被分割成区域，并由区域服务器提供服务。区域被列族垂直分为“Stores”。Stores被保存在HDFS文件。下面显示的是HBase的结构。

注意：术语“store”是用于区域来解释存储结构。
![](/images/BigData/hbase-detail-architecture.png)

HBase有三个主要组成部分：客户端库，主服务器和区域服务器。区域服务器可以按要求添加或删除。

**1 主服务器**

	-	分配区域给区域服务器并在Apache ZooKeeper的帮助下完成这个任务。
	-	处理跨区域的服务器区域的负载均衡。它卸载繁忙的服务器和转移区域较少占用的服务器。
	-	通过判定负载均衡以维护集群的状态。
	-	负责模式变化和其他元数据操作，如创建表和列。

**2	区域**	

	-区域只不过是表被拆分，并分布在区域服务器。

**3	区域服务器**	
	
	-	与客户端进行通信并处理数据相关的操作。
	-	句柄读写的所有地区的请求。
	-	由以下的区域大小的阈值决定的区域的大小。
	
	需要深入探讨区域服务器：包含区域和存储，如下图所示：
	![](/images/BigData/hbase-detail-regionServer.png)
	存储包含内存存储和HFiles。memstore就像一个高速缓存。在这里开始进入了HBase存储。数据被传送并保存在Hfiles作为块并且memstore刷新。
	