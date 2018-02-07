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

**主服务器**

	-分配区域给区域服务器并在Apache ZooKeeper的帮助下完成这个任务。
	-处理跨区域的服务器区域的负载均衡。它卸载繁忙的服务器和转移区域较少占用的服务器。
	-通过判定负载均衡以维护集群的状态。
	-负责模式变化和其他元数据操作，如创建表和列。

**区域**	

	-区域只不过是表被拆分，并分布在区域服务器。

**区域服务器**	
	
	-与客户端进行通信并处理数据相关的操作。
	-句柄读写的所有地区的请求。
	-由以下的区域大小的阈值决定的区域的大小。
	
	需要深入探讨区域服务器：包含区域和存储，如下图所示：
	
![](/images/BigData/hbase-detail-regionServer.png)

存储包含内存存储和HFiles。memstore就像一个高速缓存。在这里开始进入了HBase存储。数据被传送并保存在Hfiles作为块并且memstore刷新。 
	
**Zookeeper**	

	-Zookeeper管理是一个开源项目，提供服务，如维护配置信息，命名，提供分布式同步等
	-Zookeeper代表不同区域的服务器短暂节点。主服务器使用这些节点来发现可用的服务器。
	-除了可用性，该节点也用于追踪服务器故障或网络分区。
	-客户端通过与zookeeper区域服务器进行通信。
	-在模拟和独立模式，HBase由zookeeper来管理。
	
## Hbase Shell 
	
#### 通用命令

	status: 提供HBase的状态，例如，服务器的数量。
	
	version: 提供正在使用HBase版本。
	
	table_help: 表引用命令提供帮助。
	
	whoami: 提供有关用户的信息。	
	
#### 数据定义语言
	create: 创建一个表。
	list: 列出HBase的所有表。
	disable: 禁用表。
	is_disabled: 验证表是否被禁用。
	enable: 启用一个表。
	is_enabled: 验证表是否已启用。
	describe: 提供了一个表的描述。
	alter: 改变一个表。
	exists: 验证表是否存在。
	drop: 从HBase中删除表。
	drop_all: 丢弃在命令中给出匹配“regex”的表。
	Java Admin API: 在此之前所有的上述命令，Java提供了一个通过API编程来管理实现DDL功能。
	在这个org.apache.hadoop.hbase.client包中有HBaseAdmin和HTableDescriptor 
	这两个重要的类提供DDL功能。
	
#### 数据操纵语言
	put: 把指定列在指定的行中单元格的值在一个特定的表。
	get: 取行或单元格的内容。
	delete: 删除表中的单元格值。
	deleteall: 删除给定行的所有单元格。
	scan: 扫描并返回表数据。
	count: 计数并返回表中的行的数目。
	truncate: 禁用，删除和重新创建一个指定的表。
	Java client API: 在此之前所有上述命令，Java提供了一个客户端API来实现DML功能，
	CRUD（创建检索更新删除）操作更多的是通过编程，在org.apache.hadoop.hbase.client包下。 	在此包HTable 的 Put和Get是重要的类。
	
#### 启动 HBase Shell
要访问HBase shell，必须导航进入到HBase的主文件夹。

	cd /usr/localhost/
	cd Hbase
	
可以使用“hbase shell”命令来启动HBase的交互shell，如下图所示。

	./bin/hbase shell
	
如果已成功在系统中安装HBase，那么它会给出 HBase shell 提示符，如下图所示。
	
	HBase Shell; enter 'help<RETURN>' for list of supported commands.
	Type "exit<RETURN>" to leave the HBase Shell
	Version 0.94.23, rf42302b28aceaab773b15f234aa8718fff7eea3c, Wed Aug27
	00:54:09 UTC 2014
	hbase(main):001:0>
要退出交互shell命令，在任何时候键入 exit 或使用<Ctrl + C>。进一步处理检查shell功能之前，使用 list 命令用于列出所有可用命令。list是用来获取所有HBase 表的列表。首先，验证安装HBase在系统中使用如下所示。
	
	hbase(main):001:0> list
当输入这个命令，它给出下面的输出。

	hbase(main):001:0> list
	TABLE
	
## Hbase Admin API
HBase是用Java编写的，因此它提供Java API和HBase通信。 Java API是与HBase通信的最快方法。下面给出的是引用Java API管理，涵盖用于管理表的任务。

#### HBaseAdmin类

HBaseAdmin是一个类表示管理。这个类属于org.apache.hadoop.hbase.client包。使用这个类，可以执行管理员任务。使用Connection.getAdmin()方法来获取管理员的实例。

#####方法及说明
	1. void createTable(HTableDescriptor desc) 
		创建一个新的表
	2. void createTable(HTableDescriptor desc, byte[][] splitKeys)
		创建一个新表使用一组初始指定的分割键限定空区域
	3. void deleteColumn(byte[] tableName, String columnName)
		从表中删除列
	4. void deleteColumn(String tableName, String columnName)
		删除表中的列
	5. void deleteTable(String tableName)
		删除表
   
#### Descriptor类
#####构造函数	
						
	HTableDescriptor(TableName name)
	构造一个表描述符指定TableName对象。
#####方法及说明

	HTableDescriptor addFamily(HColumnDescriptor family)
	将列家族给定的描述符

## HBase创建表
可以使用命令创建一个表，在这里必须指定表名和列族名。在HBase shell中创建表的语法如下所示。

	create ‘<table name>’,’<column family>’
	
#### 示例
下面给出的是一个表名为emp的样本模式。它有两个列族：“personal data”和“professional data”。
|   Row key      | personal data    | professional data |
|----------------|--------------|----------------------|
|      |  |            |
|   |  |  |

在HBase shell创建该表如下所示。

	hbase(main):002:0> create 'emp', 'personal data', ’professional data’
它会给下面的输出。
	
	0 row(s) in 1.1300 seconds

	=> Hbase::Table - emp

#### 验证创建
可以验证是否已经创建，使用 list 命令如下所示。在这里，可以看到创建的emp表。

	hbase(main):002:0> list


	TABLE 

	emp
	2 row(s) in 0.0340 seconds

#### 使用Java API创建一个表
可以使用HBaseAdmin类的createTable()方法创建表在HBase中。这个类属于org.apache.hadoop.hbase.client 包。下面给出的步骤是来使用Java API创建表在HBase中。

#####第1步：实例化HBaseAdmin
这个类需要配置对象作为参数，因此初始实例配置类传递此实例给HBaseAdmin。
	
	Configuration conf = HBaseConfiguration.create();
	HBaseAdmin admin = new HBaseAdmin(conf);
#####第2步：创建TableDescriptor
HTableDescriptor类是属于org.apache.hadoop.hbase。这个类就像表名和列族的容器一样。
	
	//creating table descriptor
	HTableDescriptor table = new HTableDescriptor(toBytes("Table name"));
	//creating column family descriptor
	HColumnDescriptor family = new HColumnDescriptor(toBytes("column 	family"));
	//adding coloumn family to HTable
	table.addFamily(family);
#####第3步：通过执行管理
使用HBaseAdmin类的createTable()方法，可以在管理模式执行创建的表。
	
	admin.createTable(table);
	
下面给出的是完整的程序，通过管理员创建一个表。

	import java.io.IOException;
	
	import org.apache.hadoop.hbase.HBaseConfiguration;
	import org.apache.hadoop.hbase.HColumnDescriptor;
	import org.apache.hadoop.hbase.HTableDescriptor;
	import org.apache.hadoop.hbase.client.HBaseAdmin;
	import org.apache.hadoop.hbase.TableName;
	
	import org.apache.hadoop.conf.Configuration;
	
	public class CreateTable {
	      
	   public static void main(String[] args) throws IOException {
	
	   // Instantiating configuration class
	   Configuration con = HBaseConfiguration.create();
	
	   // Instantiating HbaseAdmin class
	   HBaseAdmin admin = new HBaseAdmin(con);
	
	   // Instantiating table descriptor class
	   HTableDescriptor tableDescriptor = new
	   TableDescriptor(TableName.valueOf("emp"));
	
	   // Adding column families to table descriptor
	   tableDescriptor.addFamily(new HColumnDescriptor("personal"));
	   tableDescriptor.addFamily(new HColumnDescriptor("professional"));
	
	
	   // Execute the table through admin
	   admin.createTable(tableDescriptor);
	   System.out.println(" Table created ");
	   }
	  }
下面列出的是输出：

	Table created
























