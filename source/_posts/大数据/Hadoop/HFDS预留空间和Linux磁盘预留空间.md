---
title: HFDS预留空间和Linux磁盘预留空间
date: 2018-11-09 16:48:05
tags: [Hadoop]
categories: [大数据]
---

组内同学反馈Spark作业提交、写数据失败，日志如下：

```js
could only be written to 8 of 10 the required nodes for XXX .
There are 254 datanode(s) running and no node(s) are excluded in this operation.
```
联系运维说是数据迁移导致1/3老服务器磁盘存储已满，并且DataNode写数据失败。
想到HDFS有预留磁盘机制（dfs.datanode.du.reserved）,磁盘写满不应该导致DataNode直接退出啊。所以在页面看了下生产机器的配置情况，发现：
![](https://raw.githubusercontent.com/xuemin-zhang/blog_resource/master/pic/Xnip2018-11-10_15-38-40.jpg)
原来是运维配置错误（说是本想要配置为200G）。
看了下磁盘物理大小，发下配置200G也有风险，因为Linux系统中，ext文件系统会默认预留5%的磁盘空间，即4T左右的盘是200G左右，如果HDFS dfs.datanode.du.reserved 参数值小于这个值其实还是会有DataNode写入失败的风险（HFDS认为还有空间可写数据，但Linux系统任务磁盘已满），所以最好是大于磁盘大小*5%一些。
