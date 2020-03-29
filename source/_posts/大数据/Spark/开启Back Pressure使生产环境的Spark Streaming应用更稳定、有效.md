---
title: 开启Back Pressure使生产环境的Spark Streaming应用更稳定、有效
date: 2017-05-29 16:48:05
copyright:
tags: [Spark]
categories: [大数据]
---
为了Spark Streaming应用能在生产中稳定、有效的执行，每批次数据处理时间（批处理时间）必须非常接近批次调度的时间间隔（批调度间隔），并且要一直低于批调度间隔。如果批处理时间一直高于批调度间隔，调度延迟就会一直增长并且不会恢复。最终，Spark Streaming应用会变得不再稳定。另一方面，如果批处理时间长时间远小于批调度间隔，就会浪费集群资源。

当Spark Streaming与Kafka使用Direct API集群时，我们可以很方便的去控制最大数据摄入量--通过一个被称作spark.streaming.kafka.maxRatePerPartition的参数。根据文档描述，他的含义是：Direct API读取每一个Kafka partition数据的最大速率（每秒读取的消息量）。

配置项spark.streaming.kafka.maxRatePerPartition，对防止流式应用在下边两种情况下出现流量过载时尤其重要：
1. Kafka Topic中有大量未处理的消息，并且我们设置是Kafka auto.offset.reset参数值为smallest，他可以防止第一个批次出现数据流量过载情况。
2. 当Kafka 生产者突然飙升流量的时候，他可以防止批次处理出现数据流量过载情况。

但是，配置Kafka每个partition每批次最大的摄入量是个静态值，也算是个缺点。随着时间的变化，在生产环境运行了一段时间的Spark Streaming应用，每批次每个Kafka partition摄入数据最大量的最优值也是变化的。有时候，是因为消息的大小会变，导致数据处理时间变化。有时候，是因为流计算所使用的多租户集群会变得非常繁忙，比如在白天时候，一些其他的数据应用（例如Impala/Hive/MR作业）竞争共享的系统资源时（CPU/内存/网络/磁盘IO）。

背压机制可以解决该问题。背压机制是呼声比较高的功能，他允许根据前一批次数据的处理情况，动态、自动的调整后续数据的摄入量，这样的反馈回路使得我们可以应对流式应用流量波动的问题。

Spark Streaming的背压机制是在Spark1.5版本引进的，我们可以添加如下代码启用改功能：
````
sparkConf.set("spark.streaming.backpressure.enabled",”true”)
````

那应用启动后的第一个批次流量怎么控制呢？因为他没有前面批次的数据处理时间，所以没有参考的数据去评估这一批次最优的摄入量。在Spark官方文档中有个被称作spark.streaming.backpressure.initialRate的配置，看起来是控制开启背压机制时初始化的摄入量。其实不然，该参数只对receiver模式起作用，并不适用于direct模式。推荐的方法是使用spark.streaming.kafka.maxRatePerPartition控制背压机制起作用前的第一批次数据的最大摄入量。我通常建议设置spark.streaming.kafka.maxRatePerPartition的值为最优估计值的1.5到2倍，让背压机制的算法去调整后续的值。请注意，spark.streaming.kafka.maxRatePerPartition的值会一直控制最大的摄入量，所以背压机制的算法值不会超过他。

另一个需要注意的是，在第一个批次处理完成前，紧接着的批次都将使用spark.streaming.kafka.maxRatePerPartition的值作为摄入量。通过Spark UI可以看到，批次间隔为5s，当批次调度延迟31秒时候，前7个批次的摄入量是20条记录。直到第八个批次，背压机制起作用时，摄入量变为5条记录。

翻译：http://www.linkedin.com/pulse/enable-back-pressure-make-your-spark-streaming-production-lan-jiang/
