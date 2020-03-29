---
title: 深入浅出Parquet Schema
date: 2016-07-29 16:48:05
tags: [其他]
categories: [其他]
---

背景

相信使用Spark比较多的人，对Parquet都不陌生。






其是Dremel的开源实现，作为一种列式存储文件格式，2015年称为 Apache 顶级项目，后来被 Spark 项目吸收，作为 Spark 的默认数据源，在不指定读取和存储格式时，默认读写 Parquet 格式的文件。


2010年 google 发表了一篇论文《Dremel: Interactive Analysis of Web-Scale Datasets》
https://ai.google/research/pubs/pub36632

A key strength of Parquet is its ability to store data that has a deeply nested structure in true columnar fashion.The result is that even nested fields can be read independently of other fields, resulting in significant performance improvements.

Another feature of Parquet is the large number of tools that support it as a format.MapReduce, Pig, Hive, Cascading, Crunch, and Spark




，介绍了其 Dermel 系统是如何利用列式存储管理嵌套数据的，嵌套数据就是层次数据，如定义一个班级，班级由同学组成，同学的信息有学号、年龄、身高等。



今天不介绍嵌套数据是如何映射到每一列了，简单来说就是把不同层级的属性拍到一级，类似降维打击。这样，一个嵌套数据可以看成独立的多个属性，每一个属性就是一列，和表结构差不多。

写流程
虽然是按列存储，但数据是一行一行来的，那什么时候将内存中的数据写文件呢？我们知道文件只能顺序写，假如每收到一行数据就写入磁盘，那就是行式存储了。

一个解决方案是为每个列开一个文件，假如数据有 n 个属性，就需要 n 个文件，每次写数据就需要追加到 n 个文件中。但是对于文件格式来说，用户肯定希望把复杂的数据存到一个文件中，而不希望管理一堆小文件（可以想象你做了一个ppt，每一页存成了一个文件），所以一个 Parquet 文件中必须存储数据的所有属性。

另一个解决方案是在内存中缓存一些数据，等缓存到一定量后，将各个列的数据放在一起打包，这样各个包就可以按一定顺序写到一个文件中。这就是列式存储的精髓：按列缓存打包。

文件格式
按照上边这种方式，Parquet 在每一列内也需要分成一个个的数据包，这个数据包就叫 Page，Page 的分割标准可以按数据点数（如每1000行数据打成一个 Page），也可以按空间占用（如每列的数据攒到8KB合成一个 Page）。

一个 Page 的数据就是一列，类型相同，在存储到磁盘之前一般都会进行编码压缩，为了快速查询、也为了解压缩这一个 Page，在写的时候先统计一下最大最小值，叫做 PageHeader，存储在 Page 的开头，其实就是 Page 的 元数据（metadata）。PageHeader 后边就是数据了，读取一个 Page 时，可以先通过 PageHeader 进行过滤。

Parquet 又把多个 Page 放在一起存储，叫 Column Chunk。于是，每一列都由多个 Column Chunk 组成，并且也有其对应的 ColumnChunk Metadata。注意，这只是一个完整数据的一个属性，一个数据的多个属性要放在多个 Column Chunk 的，这多个 Column Chunk 放在一起就叫做一个 Row Group。

下边这就是 Parquet 官方介绍：
-



Parquet文件在磁盘上的分布情况如图5所示。所有的数据被水平切分成Rowgroup，一个Row group包含这个Rowgroup对应的区间内的所有列的column chunk。一个columnchunk负责存储某一列的数据，这些数据是这一列的Repetition levels, Definition levels和values（详见后文）。一个columnchunk是由Page组成的，Page是压缩和编码的单元，对数据模型来说是透明的。一个Parquet文件最后是Footer，存储了文件的元数据信息和统计信息。Row group是数据读写时候的缓存单元，所以推荐设置较大的Rowgroup从而带来较大的并行度，当然也需要较大的内存空间作为代价。一般情况下推荐配置一个Row group大小1G，一个HDFS块大小1G，一个HDFS文件只含有一个块[s2]。




#### protobuff
Google Protocol Buffer( 简称 Protobuf) 是 Google 公司内部的混合语言数据标准。
Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

我们可以考察 Protobuf 序列化后的信息内容。您可以看到 Protocol Buffer 信息的表示非常紧凑，这意味着消息的体积减少，自然需要更少的资源。比如网络上传输的字节数更少，需要的 IO 更少等，从而提高性能。


由于文本并不适合用来描述数据结构，所以 Protobuf 也不适合用来对基于文本的标记文档（如 HTML）建模。另外，由于 XML 具有某种程度上的自解释性，它可以被人直接读取编辑，在这一点上 Protobuf 不行，它以二进制的方式存储，除非你有 .proto 定义，否则你没法直接读出 Protobuf 的任何内容


#### 列式存储：
* 按列存，能够更好地压缩数据，因为一列的数据一般都是同质的（homogenous）. 对于hadoop集群来说，空间节省非常可观.
* I/O 会大大减少，因为扫描（遍历/scan）的时候，可以只读其中部分列. 而且由于数据压缩的更好的缘故，IO所需带宽也会减小.
* 由于每列存的数据类型是相同的，we can use encodings better suited to the modern processors’ pipeline by making instruction branching more predictable. （没想好怎么翻译，各位自己理解吧）



https://www.cnblogs.com/ulysses-you/p/7985240.html
https://blog.csdn.net/macyang/article/details/8566105
