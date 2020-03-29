---
title: Kafka数据同步(镜像)工具:kafka mirror maker
date: 2016-04-02 16:48:05
tags: [Kafka]
categories: [大数据]
---

### 背景
公司数据收集后会写入kafka集群，近期涉及到机房搬迁，在完成机房搬迁移前，两个机房都有业务需要某些topic的数据，两种处理方案：1是数据写入时候双写 2是老机房数据写入完成后再同步至新机房kafka集群。本文介绍kafka自带的集群镜像工具MirrorMaker，实现kafka集群间的数据同步。

##### 一、概括来说MirrorMaker就是kafka生产者与消费者的一个整合，通过consumer从源Kafka集群消费数据，然后通过producer将数据重新推送到目标Kafka集群，如下图：



##### 二、MirrorMaker的使用相对也比较简单，下面说下启动命令及相关配置

启动脚本在$KAFKA_HOME/bin目录下，可通过命令kafka-run-class.sh kafka.tools.MirrorMaker查看相关说明：





##### 说明：
whitelist、blacklist：该工具可以同步源集群所有的或者部分topic，可以用白名单描述要同步的topic，用黑名单描述不需要同步的topic，多个topic直接逗号分隔，并且支持通配符(java http://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)
consumer.config：配置源kafka集群消费者相关信息

````
zookeeper.connect=zk1ip1:2181,zk1ip2:2181/kafka/
group.id=mirrorMaker
````

producer.config :配置目标kafka集群生产者相关信息
````
metadata.broker.list=b1:9092,b2:9092
compression.codec=none
````

启动命令：
````
sh $KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.MirrorMaker --consumer.config $KAFKA_HOME/config/mirrorMakerConsumer.config --num.streams 2 --producer.config $KAFKA_HOME/config/amirrorMakerProducer.config  —num.producers 2 --whitelist="topic2mirror"
````
