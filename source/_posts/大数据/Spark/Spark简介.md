---
title: Spark简介
date: 2016-03-29 16:48:05
tags: [其他]
categories: [其他]
---

## 历史背景

Spark由2009加州大学伯克利分校AMP实验室(UC Berkeley AMP(Algorithms, Machines, and People Lab) lab)）所开发，并与2010年开源的，类似Hadoop MapReduce的通用、并行计算框架。Spark与MapReduce最大的不同是Job的中间输出和结果可以保存在内存中，从而减少读写HDFS的开销，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代计算的场景。Spark在2013年6月进入Apache，进入高速发展期，第三方开发者贡献了大量的代码，活跃度非常高，孵化8个月后成为Apache顶级项目。

## 主要特点

* 运行速度快
Spark拥有DAG执行引擎，支持在内存中对数据进行迭代计算。官方提供的数据表明，如果数据由磁盘读取，速度是Hadoop MapReduce的10倍以上，如果数据从内存中读取，速度可以高达100倍以上。
![](http://spark.apache.org/images/logistic-regression.png)


* 使用方便
Spark支持Scala、Java、Python等语言进行编写应用程序，支持SQL进行数据处理。特别是Scala，其是一种高效、可拓展的语言，能够用简洁的代码处理较为复杂的处理工作。


* 通用性强
Spark提供了完整而强大的组件,提供批量计算、流式计算、图计算、机器学习等能力。
![](https://raw.githubusercontent.com/xuemin-zhang/blog_resource/master/pic/Xnip2018-11-19_23-18-20.jpg)


* 多执行环境
Spark具有很强的适应性，能够以HDFS、Cassandra、HBase、S3和Techyon等为持久层的读写源，能够以Mesos、YARN和自身携带的Standalone模式作为资源管理器调度、执行j任务。
<img src="http://spark.apache.org/images/spark-runs-everywhere.png" alt="Sample"  width="250" height="140">

## 包含组件
* Spark Core
  * Spark引入了RDD (Resilient Distributed Dataset) 的概念，它是分布在一些节点中的只读数据对象集合，如果数据集一部分丢失，则可以根据“血统”对它们进行重建，保证了数据的高容错性。
  * 提供了有向无环图（DAG）的分布式并行计算框架，并提供Cache机制来支持多次迭代计算或者数据共享，大大减少迭代计算之间读取数据局的开销，这对于需要进行多次迭代的数据挖掘和分析性能有很大提升。
  * 移动计算而非移动数据，RDD Partition可以就近读取分布式文件系统中的数据块到各个节点内存中进行计算。
  * 可以使用多线程池模型来减少task启动开稍。
  * 采用容错的、高可伸缩性的akka作为通讯框架。


* Spark SQL
  * 是一个可以与Spark Core一起使用的组件，支持使用SQL处理结构化的数据，RDD、Hive数据。引入了新的RDD类型SchemaRDD（DataFram），可以象传统数据库定义表一样来定义SchemaRDD。SchemaRDD可以从RDD转换过来，也可以从Parquet文件读入，也可以使用HiveQL从Hive中获取。
  * 内嵌了Catalyst查询优化框架，在把SQL解析成逻辑执行计划之后，利用Catalyst包里的一些类和接口，执行了一些简单的执行计划优化，最后变成RDD的计算
  * 在应用程序中可以混合使用不同来源的数据，如可以将来自HiveQL的数据和来自RDD的数据进行Join操作。


* Spark Streaming
  * 是一个对实时数据进行流式处理的、高容错、高吞吐量的系统，可以对接Kafka、Flume、TCP等多种类型数据源，并对其进行类似Map、Reduce和Join等复杂操作，并支持将结果存储到HDFS、S3等存储文件系统，HBase、MySQL等数据库或应用到实时仪表盘。
  * 并不是真正意义上的流式处理，而是微批，也是对Spark Core核心APId扩展，会将处理的数据流抽象为Dstream，DStream本质上表示RDD的序列，所以任何对DStream的操作都会转变为对底层RDD的操作。



* MLlib
Spark目前有两个机器学习包，ML和MLlib。 MLlib是Spark的机器学习组件之一，是一套用Spark编写的机器学习和统计算法。 Spark ML仍处于早期阶段，但自Spark 1.2以来，它提供了比MLlib更高级别的API，可帮助用户更轻松地创建实用的机器学习管道。 Spark MLLib主要建立在RDD之上，而ML建立在SparkSQL数据框之上。 4最终，Spark社区计划转向ML并弃用MLlib。 Spark ML和MLLib有一些独特的性能考虑因素，特别是在处理大数据量和高速缓存时，我们在???中介绍了一些。



* GraphX
是一个构建在Spark之上的图形处理框架，带有用于图形计算的API。 Graph X是Spark中最不成熟的组件之一，因此我们不会详细介绍它。 在Spark的未来版本中，类型图功能将开始在数据集API之上引入。 我们将粗略地看一下???中的图X.

GraphX是Spark中用于图(e.g., Web-Graphs and Social Networks)和图并行计算(e.g., PageRank and Collaborative Filtering)的API,可以认为是GraphLab(C++)和Pregel(C++)在Spark(Scala)上的重写及优化，跟其他分布式图计算框架相比，GraphX最大的贡献是，在Spark之上提供一栈式数据解决方案，可以方便且高效地完成图计算的一整套流水作业。GraphX最先是伯克利AMPLAB的一个分布式图计算框架项目，后来整合到Spark中成为一个核心组件。

1.对Graph视图的所有操作，最终都会转换成其关联的Table视图的RDD操作来完成。这样对一个图的计算，最终在逻辑上，等价于一系列RDD的转换过程。因此，Graph最终具备了RDD的3个关键特性：Immutable、Distributed和Fault-Tolerant。其中最关键的是Immutable（不变性）。逻辑上，所有图的转换和操作都产生了一个新图；物理上，GraphX会有一定程度的不变顶点和边的复用优化，对用户透明。

2.两种视图底层共用的物理数据，由RDD[Vertex-Partition]和RDD[EdgePartition]这两个RDD组成。点和边实际都不是以表Collection[tuple]的形式存储的，而是由VertexPartition/EdgePartition在内部存储一个带索引结构的分片数据块，以加速不同视图下的遍历速度。不变的索引结构在RDD转换过程中是共用的，降低了计算和存储开销。

3.图的分布式存储采用点分割模式，而且使用partitionBy方法，由用户指定不同的划分策略（PartitionStrategy）。划分策略会将边分配到各个EdgePartition，顶点Master分配到各个VertexPartition，EdgePartition也会缓存本地边关联点的Ghost副本。划分策略的不同会影响到所需要缓存的Ghost副本数量，以及每个EdgePartition分配的边的均衡程度，需要根据图的结构特征选取最佳策略。目前有EdgePartition2d、EdgePartition1d、RandomVertexCut和CanonicalRandomVertexCut这四种策略。在淘宝大部分场景下，EdgePartition2d效果最好。


参考：http://spark.apache.org/
