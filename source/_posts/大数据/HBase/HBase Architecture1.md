---
title: HBase Architecture
date: 2014-12-13 16:48:05
tags: [HBase]
categories: [大数据]
---

## 65. 概述
### 65.1. NoSQL?
HBase是一种“NoSQL”数据库。 “NoSQL”是一个通用术语，意味着数据库不是以SQL作为主要访问语言的RDBMS，但是有许多类型的NoSQL数据库：BerkeleyDB是单实例NoSQL数据库的一个例子，而HBase是分布数据库。 从技术上讲，HBase实际上更像是“数据存储”而不是“数据库”，因为它缺少许多关系型数据库的特性，，例如类型列，二级索引，触发器和高级查询语言等。

HBase集群扩展通过增加RegionServer来实现。如果一个集群从10扩展到20个RegionServer，那么，不仅仅是存储容量增加一倍，连处理能力也会增加一倍。对于关系型数据库而言，也可以很好的扩展，但只能达到一定的程度（单个数据库服务器的能力），并且为了获得最佳性能，需要专门的硬件和存储设备。HBase特性如下：

* 强一致性读写：HBase不是一个“最终一致性”的数据存储。这使得它更适合高速度的聚集任务。
* 自动分区：HBase的表通过region被分布在集群中，而region是自动拆分并重新分布数据行的。
* 自动RegionServer容灾
* Hadoop/HDFS集成：HBase支持HDFS作为它的分布式文件系统
* MapReduce：HBase支持通过MapReduce基于HBase作为数据源的大量的并行处理
* Java Client API：HBase支持通过Java API编程的方式来访问
* Thrift/REST API：HBase也支持Thrift和REST这样的非Java的客户端
* Block Cache and Bloom Filters：对于大容量查询优化， HBase支持 Block Cache 和 Bloom Filters
* Operational Management：HBase提供web界面

### 65.2. 什么时候适合使用HBase

第一、确信你有足够的数据。如果你有数以亿计的数据行，那么HBase是一个不错的选择。如果你只有数千或者百万的数据，那么使用传统的关系型数据库可能更好，因为事实上你的这些数据可能只需要一个或者两个节点就能处理得完，这样的话集群中的其它的节点就处于空闲状态。

第二、确保你不需要用到关系型数据库的特性（比如：固定类型的列、辅助索引、事务、查询语言等等）。基于关系型数据库构建的应用不能通过简单的改变JDBC驱动来传输到HBase中。从RDBMS到HBase是完全相反的两套设计。

第三、确保你有足够的硬件。因为当DataNode数量小于5的时候HDFS将不能很好的工作了。

### 65.3. HBase 和 Hadoop/HDFS 的区别?
HDFS是一个分布式的文件系统，适合存储大文件，但它不能提供快速的个性化的在文件中查找。HBase是构建于HDFS基础之上的，并且它支持对大表的中的记录进行快速查找和更新。HBase内部将数据存放在HDFS中被索引的“StoreFiles”上以供快速查找。

## 66. Catalog Tables
目录表 -ROOT- 和 .META. 作为 HBase 表存在。他们被HBase shell的 list 命令过滤掉了， 但他们和其他表一样存在。
### 66.1 ROOT
-ROOT- 保存 .META. 表存在哪里的踪迹. -ROOT- 表结构如下:

Key:
* .META. region key (.META.,,1)

Values:
* info:regioninfo (序列化.META.的 HRegionInfo 实例 )
* info:server ( 保存 .META.的RegionServer的server:port)
* info:serverstartcode ( 保存 .META.的RegionServer进程的启动时间)

### 66.2 META
.META. 保存系统中所有region列表。 .META.表结构如下:

Key:
* Region key 格式 ([table],[region start key],[region id])

Values:
* info:regioninfo (序列化.META.的 HRegionInfo 实例 )
* info:server ( 保存 .META.的RegionServer的server:port)
* info:serverstartcode ( 保存 .META.的RegionServer进程的启动时间)

当表在分割过程中，会创建额外的两列, info:splitA 和 info:splitB 代表两个女儿 region. 这两列的值同样是序列化HRegionInfo 实例. region最终分割完毕后，这行会删除。

HRegionInfo的备注: 空 key 用于指示表的开始和结束。具有空开始键值的region是表内的首region。 如果 region 同时有空起始和结束key，说明它是表内的唯一region。

在需要编程访问(希望不要)目录元数据时，参考 Writables 工具。

## 67. 客户端
HBase客户端的 HTable类负责寻找相应的RegionServers来处理行。他是先查询 .META. 和 -ROOT 目录表。然后再确定region的位置。定位到所需要的区域后，客户端会直接 去访问相应的region(不经过master)，发起读写请求。这些信息会缓存在客户端，这样就不用每发起一个请求就去查一下。如果一个region已经废弃(原因可能是master load balance或者RegionServer死了)，客户端就会重新进行这个步骤，决定要去访问的新的地址。

## 68. 客户端请求过滤器
Get 和 Scan 实例可以用 filters 配置，以应用于 RegionServer。
过滤器可能会搞混，因为有很多类型的过滤器，最好通过理解过滤器功能组来了解他们。

## 69. Master
HMaster是Master Server的一个实现。Master Server负责监视集群中所有的RegionServer实例，并且它也是所有元数据改变的一个对外接口。在分布式集群中，典型的Master运行在NameNode那台机器上。

### 69.3. Interface
HMasterInterface接口是操作元数据的主要接口，提供以下操作：
* Table (createTable, modifyTable, removeTable, enable, disable)
* ColumnFamily (addColumn, modifyColumn, removeColumn)
* Region (move, assign, unassign)

## 70. RegionServer
HRegionServer是RegionServer的实现，它负责服务并管理regions。在分布式集群中，一个RegionServer通常运行在一个DataNode上。

### 70.1. Interface
HRegionRegionInterface既包含数据的操作也包含region维护的操作

* Data (get, put, delete, next, etc.)
* Region (splitRegion, compactRegion, etc.)

### 70.5. RegionServer Splitting Implementation
region server处理写请求，它们被累积在内存中一个叫memstore的地方。一旦memstore文件满了，内容将被写到磁盘上作为store file。这个事件叫做memstore flush。随着store file的不断累积，RegionServer将合并它们成大文件，以减少store file的数量。在每次刷新或者合并以后，region中数据的数量会发生改变。RegionServer根据切分策略来查看是否region太大了或者应该被切分。

逻辑上，region切分的操作很简单。找一个合适的位置，将region中的数据切分成两个新的region。然而，这个处理的过程并不简单。当切分发生的时候，数据并不是立刻被重写到这个心创建的女儿region上。
