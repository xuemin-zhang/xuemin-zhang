---
title: HDFS小文件处理--文件归档Archive
date: 2015-04-02 16:48:05
tags: [Kafka]
categories: [大数据]
---
Hadoop并不擅长对小型文件的储存，原因取决于Hadoop文件系统的文件管理机制，Hadoop的文件存储的单元为一个块（block），block的数据存放在集群中的datanode节点上，由namenode对所有datanode存储的block进行管理。namenode将所有block的元数据存放在内存中，以方便快速的响应客户端的请求。那么问题来了，不管一个文件有多小，Hadoop都把它视为一个block，大量的小文件，将会把namenode的内存耗尽。
那么如何对大量的小文件进行有效的处理呢？Hadoop的优秀工程师们其实已经为我们考虑好了，Hadoop提供了一个叫Archive归档工具，Archive可以把多个文件归档成为一个文件，换个角度来看，Archive实现了文件的元数据整理，但是，归档的文件大小其实没有变化，只是压缩了文件的元数据大小。

## 归档小文件
使用命令：
``
hadoop archive -archiveName test.har -p /user/archive/test /user/har
``

参数含义：
-archiveName 指定归档文件名；
-p    指定要进行归档目录的父目录，支持同时归档多个子目录；
/user/archive/test    归档文件的目录
/user/har    归档文件存放的目录


## 查看归档文件
har文件采用一套不同于hdfs的路径协议。基本格式为：har://<hdfs路径>。在未添加协议url时，我们看到的是har文件在HDFS中的底层形式，加上协议头后我们可以看到其目录结构与源目录结构相同。
``
hadoop fs -ls -R har:///user/har/test.har
``

## 归档的不足
1、archive不支持压缩，所以归档前后占用空间没有减少；
2、archive一旦创建就不能进行修改；
3、archive虽然解决了namenode的空间问题，但是，在执行mapreduce时，会把多个小文件交给同一个mapreduce去split，这会降低mapreduce的效率。

## 解除归档
如果需要修改文件或者想提升mapreduce处理的效率，可以考虑将数据解除归档。
`
hadoop distcp har:///user/har/test.har /user/tmp/test
`
