---
title: HBase0.96 升级步骤及源代码分析
date: 2014-11-12 16:48:05
tags: [HBase]
categories: [大数据]
---

测试的升级环境为cdh4.3 升级到社区版 Hadoop2.2/HBase0.96。

### 一、验证HDFS和Zookeeper已正常运行
（HDFS and ZooKeeper must be up!）



### 二、在集群中任一服务器上执行
hbase upgrade -check

1.下面是该命令的输出（官网给的例子，自己执行的时候输出不同，由于官网的有HFileV1的文件，故用之举例）
````
Tables Processed:
hdfs://localhost:41020/myHBase/.META.
hdfs://localhost:41020/myHBase/usertable
hdfs://localhost:41020/myHBase/TestTable
hdfs://localhost:41020/myHBase/t

Count of HFileV1: 2
HFileV1:
hdfs://localhost:41020/myHBase/usertable    /fa02dac1f38d03577bd0f7e666f12812/family/249450144068442524
hdfs://localhost:41020/myHBase/usertable    /ecdd3eaee2d2fcf8184ac025555bb2af/family/249450144068442512

Count of corrupted files: 1
Corrupted Files:
hdfs://localhost:41020/myHBase/usertable/fa02dac1f38d03577bd0f7e666f12812/family/1
Count of Regions with HFileV1: 2
Regions to Major Compact:
hdfs://localhost:41020/myHBase/usertable/fa02dac1f38d03577bd0f7e666f12812
hdfs://localhost:41020/myHBase/usertable/ecdd3eaee2d2fcf8184ac025555bb2af

There are some HFileV1, or corrupt files (files with incorrect major version)
 HFileV1: 2
HFileV1:
hdfs://localhost:41020/myHBase/usertable    /fa02dac1f38d03577bd0f7e666f12812/family/249450144068442524
hdfs://localhost:41020/myHBase/usertable    /ecdd3eaee2d2fcf8184ac025555bb2af/family/249450144068442512

Count of corrupted files: 1
Corrupted Files:
hdfs://localhost:41020/myHBase/usertable/fa02dac1f38d03577bd0f7e666f12812/family/1
Count of Regions with HFileV1: 2
Regions to Major Compact:
hdfs://localhost:41020/myHBase/usertable/fa02dac1f38d03577bd0f7e666f12812
hdfs://localhost:41020/myHBase/usertable/ecdd3eaee2d2fcf8184ac025555bb2af

There are some HFileV1, or corrupt files (files with incorrect major version)
`````

2.说明：
该命令的作用是检测HBase的数据文件（HDFS）是否有HFileV1类型的（HFile V1和V2是HDFS的两个版本，文件数据组织格式有些差别）和corrupted的。HBase从0.94开始支持HFileV2类型文件，0.94是两个版本都兼容，0.96则只兼容HFileV2，所以要处理遗留的HFileV1文件，处理方法是对这些HFileV1文件进行 major compact。corrupted文件可试着修复，数量不多或者数据完整性要求不高的话可以直接move了。确保该步结果如下图，再执行下面步骤。


### 三、在集群中任一服务器上执行

hbase upgrade -execute

1.下面是该命令的输出
````
Starting Namespace upgrade
Created version file at hdfs://localhost:41020/myHBase with version=7
Migrating table testTable to hdfs://localhost:41020/myHBase/.data/default/testTable
…..
Created version file at hdfs://localhost:41020/myHBase with version=8
Successfully completed NameSpace upgrade.
Starting Znode upgrade
….
Successfully completed Znode upgrade

Starting Log splitting
…
Successfully completed Log splitting
````

2.说明：
该命令的作用是触发升级任务。从输出日志可以看出，升级涉及到三部分内容：
````
1）Namespace upgrade
2）Znode upgrade
3）Log upgrade
````

我在执行时抛出了如下异常：
````
14/11/12 11:49:05 WARN wal.HLogSplitter: Could not open hdfs://XXX/hbase/WALs/XXX,60020,1415264007965/XXX%2C60020%2C1415264007965.1415618778662 for reading. File is empty
java.io.EOFException
        at java.io.DataInputStream.readFully(DataInputStream.java:197)
        at java.io.DataInputStream.readFully(DataInputStream.java:169)
        at org.apache.hadoop.io.SequenceFile$Reader.init(SequenceFile.java:1845)
        at org.apache.hadoop.io.SequenceFile$Reader.initialize(SequenceFile.java:1810)
        at org.apache.hadoop.io.SequenceFile$Reader.<init>(SequenceFile.java:1759)
        at org.apache.hadoop.io.SequenceFile$Reader.<init>(SequenceFile.java:1773)
        at org.apache.hadoop.hbase.regionserver.wal.SequenceFileLogReader$WALReader.<init>(SequenceFileLogReader.java:69)
        at org.apache.hadoop.hbase.regionserver.wal.SequenceFileLogReader.reset(SequenceFileLogReader.java:174)
        at org.apache.hadoop.hbase.regionserver.wal.SequenceFileLogReader.initReader(SequenceFileLogReader.java:183)
        at org.apache.hadoop.hbase.regionserver.wal.ReaderBase.init(ReaderBase.java:68)
        at org.apache.hadoop.hbase.regionserver.wal.HLogFactory.createReader(HLogFactory.java:126)
        at org.apache.hadoop.hbase.regionserver.wal.HLogFactory.createReader(HLogFactory.java:89)
        at org.apache.hadoop.hbase.regionserver.wal.HLogSplitter.getReader(HLogSplitter.java:645)
        at org.apache.hadoop.hbase.regionserver.wal.HLogSplitter.getReader(HLogSplitter.java:554)
        at org.apache.hadoop.hbase.regionserver.wal.HLogSplitter.splitLogFile(HLogSplitter.java:273)
````
从日志可以看出是Log upgrade时发现WAL日志不存在（HDFS上该文WALs目录确实为空），虽然有异常，但不影响升级结果，可忽略（有点不负责任的感觉，有时间了再找找原因）。

### 四、源代码分析

具体执行升级任务的是org.apache.hadoop.hbase.migration包下的UpgradeTo96类（可从异常日志中看到，也可从两个命令跟踪到该类，有时间了再补一篇各命令脚本执行的顺序及调用类的文章），从run（）方法看起：

1、首先跟下check步骤：



可以看出相关的类是：HFileV1Detect\or，看下其run（）方法：



先看看processResult（）



再跟checkFOrV1Files（）找找hFileV1Set 和 corruptedHFiles



跟进isTableDir()可以看出其是判断Path是否为文件

 

再看processTable()



它先调了processRegion（）：





在该方法里，遍历了region中列族的storefile文件，并对其进行版本判断，版本算法见computeMajorVersion()



2、再看下upgrade步骤



还记得前文提到的升级的三部分内容吧：
````
1）Namespace upgrade
2）Znode upgrade
3）Log upgrade
````
看下NamespaceUpgrade类的run（）：



跟进init()

定义了几个重要目录（由于0.94和0.96元数据组织上有很大差别，目录组织也有很大变化，这个就不截图了）：
````
/hbase/data/deflate（该目录下放的是用户表）
/hbase/data/hbase  （该目录下放的是meta、namespace数据）
/hbase/.migration   （Any artifacts left from migration can be moved here）
````
再看upgradeTableDirs():
这个方法就是升级工作的具体内容，注释是我添上去的，就不一一跟进方法里了

1）makeNamespaceDirs() 创建 /hbase/data/hbase /hbase/data/deflate目录
2）migrateTables()  move用户表数据至/hbase/data/deflate目录，其实是调用了hdfs的rename方法
3）migrateSnapshots() move快照文件，也是rename，.snapshot -->.hbase-snapshot
4）migrateDotDirs()    Dot dirs to rename.  Leave the tmp dir named '.tmp' and snapshots as .hbase-snapshot.
5）migrateMeta()    move meta 表至 /hbase/data/hbase目录下，也是rename
6）migrateACL()     同上
7）deleteRoot()   删除root表


另外：
官网说这个升级是不可逆的，但根据升级代码看是可以的，就是备份upgradeTableDirs（）方法中处理的目录，当然用户表不用备份，待测。
