---
title: HBase数据模型(Data Model)
date: 2014-10-02 16:48:05
tags: [HBase]
categories: [大数据]
---

在HBase中，数据存储在有行有列的二维表。HBase中的表是与关系型数据库重复的术语，但这个类比对我们理解HBase来说帮助不大。将HBase的表当做一个多层的Map，反而会可以容易理解。

## HBase数据模型相关术语
### Table（表）
一个HBase表由多行组成。

#### Row（行）

HBase中的行里面包含一个行键和一个或者多个包含值的列。行按照行键的字母顺序存储在表中。因为这个原因，行键的设计就显得非常重要。数据的存储目标是相关的数据存储在一起。一个常见的行键模式是网站域名。如果你的行键是域名，你应该将域名进行反转(org.apache.www, org.apache.mail, org.apache.jira)再存储。这样的话，所有Apache域名将会存储在一起，好过基于子域名的首字母分散在各处。

#### Column（列）

HBase中的列包含用“:”分隔开的列族和列限定符。

#### Column Family（列族）

因为性能的原因，列族物理上包含一组列和它们的值。每一个列族拥有一系列的存储属性，例如值是否缓存在内存中，数据如何压缩或者他的行键是否要编码等等。表中的每一行拥有相同的列族，尽管一个给定的行可能没有存储任何数据在一个给定的列族中。

#### Column Qualifier（列限定符）

列限定符将添加到列族中，以便为给定的数据提供索引。例如给定了一个列族content，那么限定符可能是content:html，也可以是content:pdf。虽然列族在创建表时是固定的，但列限定符是可变的，并且行与行之间可能有很大差异。

#### Cell（单元）

单元是由行、列族、列限定符、值和代表值版本的时间戳组成的。

#### Timestamp（时间戳）
时间戳与每个值一起写入，并且是数据给定版本的标识符。默认情况下，时间戳表示写入数据时RegionServer上的时间，但你也可以在写入数据时指定一个不同于RegionServer上的时间的时间戳。

## 概念视图
可以读一下Jim R博客中一篇非常容易理解的解释HBase数据模型的文章Understanding HBase and BigTable。Amandeep Khurana的PDF Introduction to Basic Schema Design中提供了另一个很好的解释。

阅读不同观点的资料可能会帮助你更透彻地了解HBase的设计。链接的文章与本节中的信息具有相同的基础。

接下来的例子取自BigTable论文第二页中的例子，在此基础上做了些许的改变。一个名为webtable的表，表中有两行（com.cnn.www 和 com.example.www）和三个列族（contents, anchor, 和 people）。在这个例子当中，第一行(com.cnn.www)中列族anchor包含两列（anchor:cssnsi.com, anchor:my.look.ca），列族content包含一列（contents:html）。这个例子中com.cnn.www行拥有5个版本，而com.example.www行有一个版本。contents:html列中包含给定网页的整个HTML。anchor列族下的限定符，每个都包含链接到该行所代表的站点的外部站点，以及它在其链接的锚点中使用的文本。People列族代表与该网站相关联的人员。

**列名**
````
按照约定，一个列名的格式为列族名前缀加限定符。例如，列contents:html由列族contents和html限定符。
冒号（:）用于将列族和列限定符分开。
````

**Table webtable**

在HBase中，表中的单元如果是空将不占用空间或者事实上不存在。这就使得HBase看起来“稀疏”。表视图不是唯一方式来查看HBase中数据，甚至不是最精确的。下面的方式以多层级Map的方式来表达相同的信息。下面只是一个用于说明目的的模型，可能不是百分百的精确。
````
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
     }
     anchor: {
       t9: anchor:cnnsi.com = "CNN"
       t8: anchor:my.look.ca = "CNN.com"
     }
     people: {}
   }

  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
     }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
````

## 物理视图
尽管一个概念层次的表看起来是由一些列稀疏的行组成，但他们是通过列族来存储的。一个新建的限定符(column_family:column_qualifier)可以随时地添加到已存在的列族中。


概念视图中的空单元实际上是没有进行存储的。因此对于返回时间戳为t8的contents:html的值的请求，结果为空。同样的，一个返回时间戳为t9的anchor:my.look.ca的值的请求，结果也为空。然而，如果没有指定时间戳的话，那么会返回特定列的最新值。对有多个版本的列，优先返回最新的值，因为时间戳是按照递减顺序存储的。因此对于一个返回com.cnn.www里面所有的列的值并且没有指定时间戳的请求，返回的结果会是时间戳为t6的contents:html的值、时间戳 为t9的anchor:cnnsi.com的值和时间戳为t8的anchor:my.look.ca的值。

关于Apache Hase如何存储数据的内部细节，查看regions.arch.

## 命名空间
命名空间是类似于关系数据库系统中数据库表的逻辑分组。这个抽象的概念为即将到来的多租户相关特性奠定了基础：

* Quota Management (HBASE-8410) - Restrict the amount of resources (i.e. regions, tables) a namespace can consume.
* Namespace Security Administration (HBASE-9206) - Provide another level of security administration for tenants.
* Region server groups (HBASE-6721) - A namespace/table can be pinned onto a subset of RegionServers thus guaranteeing a course level of isolation.

### 命名空间管理
命名空间可以被创建、移除和修改。命名空间成员资格是在表创建期间通过指定表单的完全限定表名来确定的：
````
<table namespace>:<table qualifier>
````

Example 11. Examples
````
#Create a namespace
create_namespace 'my_ns'

#create my_table in my_ns namespace
create 'my_ns:my_table', 'fam'

#drop namespace
drop_namespace 'my_ns'

#alter namespace
alter_namespace 'my_ns', {METHOD => 'set', 'PROPERTY_NAME' => 'PROPERTY_VALUE'}

````
### 预定义命名空间

有两种预定义的特殊的命名空间
* hbase – 系统命名空间, 用于包含HBase内部表
* default – 没有明确指定命名空间的表将会自动落入这个命名空间

Example 12. Examples
````
#namespace=foo and table qualifier=bar
create 'foo:bar', 'fam'

#namespace=default and table qualifier=bar
create 'bar', 'fam'
````

## 22. 表
在模式定义时预先声明表。

## 23. 行
行键是未解释的字节。行是按照字典顺序进行排序的并且最小的排在前面。空的字节数据用来表示表的命名空间的开头和结尾。

## 24. 列族
列在HBase中是归入到列族里面的。一个列的所有列成员都涌向相同的前缀。例如，列courses:history和cources:math是cources列族的成员，冒号用于将列族和列限定符分开。列族前缀必须由可打印的字符组成。列限定符可以由任意字节组成。列族必须在结构定义阶段预先声明号而列则不需要再结构设计阶段预先定义而是可以在表的创建和运行阶段快速的加入。

物理存储上，所有的列族成员都存储在文件系统。由于调优和存储规范都是在列族层次上，建议所有的列族成员具有相同的通用的访问模式和大小特征。

## 25. 单元
一个{row,column,version}完全指定了HBase的一个单元。单元内容是未解释的字节

## 26. 数据模型操作
数据模型的四个主要操作是Get，Put，Scan和Delete。可以通过Table实例进行操作。

### 26.1. 获取
Get 返回指定行的属性。Gets通过Table.get.执行。

### 26.2. 插入
Put 操作是在行键不存在时添加新行或者行键已经存在时进行更新。 Puts 是通过 Table.put (写缓存) 或者Table.batch (没有写缓存)执行的。

### 26.3. 扫描
Scan 允许迭代多行以获取指定的属性。

下面是表实例中Scan的例子。假设一个表里面有"row1", "row2", "row3"，然后有另外一组行键为"abc1", "abc2",和"abc3"。下面的例子展示如何设置一个Scan实例来返回以“row”开头的行。
````Java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Table table = ...      // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
````
请注意，通常，为扫描指定特定停止点的最简单方法是使用InclusiveStopFilter类。

### 26.4. 删除
Delete 操作是将一个行从表中移除. Deletes 通过 Table.delete执行。
HBase不会立刻对数据的进行操作，而是为死亡数据创建一个称为墓碑的标签。这个墓碑和死亡数据会在“major compations”中被清除。
查看 version.delete 获取更多关于列的版本删除的信息，查看 compaction 获取关于compations的更多信息。

## 27. 版本
在HBase通过{row, column, version}完全指定一个单元。可以有无限数量的单元格，他们行和列相同，但单元格地址仅在其版本维度上有所不同。

行和列使用字节来表达，而版本是通过长整型来指定的。典型来说，这个long类型的时间实例就像java.util.Date.getTime() 或者 System.currentTimeMillis()返回的一样，以毫秒为单位，返回当前时间和January 1, 1970 UTC的时间差。

HBase的版本维度以递减顺序存储，以致读取一个存储的文件时，返回的是最新版本的数据。

在HBase中，对单元版本的语义存在很多困惑。 尤其是：
* 如果多个数据写到一个具有相同版本的单元里，只能获取到最后写入的那个
* 以非递增的版本顺序写入也是可以的。

下面我们将描述HBase中版本维度是如何运作的。可以看 HBASE-2406 关于HBase版本的讨论。 Bending time in HBase 是关于HBase的版本或者时间维度的好读物。它提供了比这里更多的关于版本的细节信息。正如这里写到的，这里提到的覆盖存在的时间戳的限制将不再存在。这部分只是Bruno Dumon所写的关于版本的基本大纲。

### 27.1.指定版本的存储数量
版本的最大存储数量是列结构的一个部分并且在表创建时指定，或者通过alter命令行，或者通过 HColumnDescriptor.DEFAULT_VERSIONS来修改。HBase0.96之前，默认数量是3，HBase0.96之后改为1.

Example 13. 修改一个列族的最大版本数.
````
这个例子使用HBase Shell来修改列族 f1的最大版本数为5，你也可以使用 HColumnDescriptor来实现。

hbase> alter ‘t1′, NAME => ‘f1′, VERSIONS => 5
````
Example 14. 修改一个列族的最小版本数Modify
````
你也可以通过指定最小半本书来存储列族。默认情况下，该值为零，意味着这个属性是禁用的。下面的例子是通过HBase Shell设置列族f1中的所有列的最小版本数为2。你也可以通过 HColumnDescriptor来实现。

hbase> alter ‘t1′, NAME => ‘f1′, MIN_VERSIONS => 2
````
从HBase0.98.2开始，你可以通过设定在hbase-site.xml中设置hbase.column.max.version属性为所有新建的列指定一个全局的默认的最大版本数。

### 27.2. 版本和HBase 操作
在这部分我们来看一下版本维度在HBase的每个核心操作中的表现。

#### 27.2.1. Get/Scan
Get是通过获取Scan的第一个数据来实现的。下面的讨论适用于 Get 和 Scans.。

默认情况下，如果你没有指定明确的版本，当你执行一个Get操作时，那个版本为最大值的单元将被返回（可能是也可能不是最新写人的那个）。默认的行为可以通过下面方式来修改：

* 返回不止一个版本 查看 Get.setMaxVersions()
* 返回最新版本以外的版本, 查看 Get.setTimeRange()

想要获得小于或等于固定值的最新版本，仅仅通过使用一个0到期望版本的范围和设置最大版本数为1就可以实现获得一个特定时间点的最新版本的记录。

#### 27.2.2. 默认的Get 例子
下面例子仅仅返回行的当前版本。
````Java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
````
#### 27.2.3. Get版本的例子
下面是获得行的最新3个版本的例子：
````Java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
get.setMaxVersions(3);  // will return last 3 versions of row
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
List<KeyValue> kv = r.getColumn(CF, ATTR);  // returns all versions of this column
````
#### 27.2.4. Put
Put操作常常是以固定的时间戳来创建一个新单元。默认情况下，系统使用服务的 currentTimeMillis，但是你也可以为每一个列自己指定版本（长整型）。这就意味着你可以指定一个过去或者未来的时间点，或者不是时间格式的长整型。

为了覆盖已经存在的值，对和那个你想要覆盖的单元完全一样的row、column和version进行put操作。

##### 隐式版本例子
下面Put是以当前时间为版本的隐式操作
````Java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put(Bytes.toBytes(row))
put.add(CF, ATTR, Bytes.toBytes( data));
table.put(put);
````
##### 显示版本例子
下面的put是显示指定时间戳的操作。
````
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put( Bytes.toBytes(row));
long explicitTimeInMs = 555;  // just an example
put.add(CF, ATTR, explicitTimeInMs, Bytes.toBytes(data));
table.put(put);
````
警告: 版本时间戳是HBase内部用来计算数据的存活时间的。它最好避免自己设置。最好是将时间戳作为行的单独属性或者作为key的一部分，或者两者都有。

#### 27.2.5. Delete

有三种不同的删除类型。可以看看Lars Hofhansl所写的博客 Scanning in HBase: Prefix Delete Marker.
* Delete:列的指定版本
* Delete column:列的所有版本
* Delete family:特定列族里面的所有列。
当要删除整个行时，HBase将会在内部为每一个列族创建一个墓碑。

删除通过创建一个墓碑标签来工作的。例如，让我们来设想我们要删除一个行。为此你可指定一个版本，或者使用默认的currentTimeMillis 。这就是删除小于等于该版本的所有单元。HBase不会修改数据，例如删除操作将不会立刻删除满足删除条件的文件。相反的，称为墓碑的会被写入，用来掩饰被删除的数据。当HBase执行一个压缩操作，墓碑将会执行一个真正地删除死亡值和墓碑自己的删除操作。如果你的删除操作指定的版本大于目前所有的版本，那么可以认为是删除整个行的数据。

你可以在 Put w/timestamp → Deleteall → Put w/ timestamp fails 用户邮件列表中查看关于删除和版本之间的相互影响的有益信息。
keyvalue也可以到keyvalue查看更多关于内部KeyValue格式的信息。

删除标签会在下一次仓库压缩操作中被清理掉，除非为列族设置了 KEEP_DELETED_CELLS (查看 Keeping Deleted Cells)。为了保证删除时间的可配置性，你可以通过在 hbase-site.xml.中hbase.hstore.time.to.purge.deletes属性来设置TTL（生存时间）。如果 hbase.hstore.time.to.purge.deletes没有设置或者设置为0，所有的删除标签包括哪些墓碑都会在下一次精简操作中被干掉。此外，未来带有时间戳的删除标签将会保持到发生在hbase.hstore.time.to.purge.deletes加上代表标签的时间戳的时间和的下一次精简操作。

### 27.3. 当前的局限性
#### 27.3.1. Deletes mask Puts删除覆盖插入/更新
删除操作覆盖插入/更新操作，即使put在delete之后执行的。可以查看 HBASE-2256. 还记得一个删除写入一个墓碑，只有当下一次精简操作发生时才会执行真正地删除操作。假设你执行了一个删除全部小于等于T的操作。在此之外又做了一个时间戳为T的put操作。这个put操作即使是发生在delete之后，也会被delete墓碑所覆盖。执行put的时候不会报错，不过当你执行一个get的时候会发现执行无效。你会在精简操作之后重新开始工作。如果你在put的使用的递增的版本，那么这些问题将不会出现。但如果你不在意时间，在执行delelte后立刻执行put的话，那么它们将有可能发生在同一时间点，这将会导致上述问题的出现。

#### 27.3.2. 精简操作影响查询结果
创建三个版本为t1,t2,t3的单元，并且设置最大版本数为2.所以当我们查询所有版本时，只会返回t2和t3。但是当你删除版本t2和t3的时候，版本t1会重新出现。显然，一旦重要精简工作运行之后，这样的行为就不会再出现。（查看 Bendingtime in HBase.）

## 28. 排序次序
HBase中所有的数据模型操作返回的数据都是经过排序的。首先是行排序，其次是列族，接着是列限定符，最后是时间戳（递减排序，最新的记录最先返回）

## 29. 列元数据
所有列的元数据都存储在一个列族的内部KeyValue实例中。因此，HBsase不仅支持一行中有多列，而且支持行之间的列的差异多样化。跟踪列名是你的责任。

唯一获取一个列族的所有列的方法是处理所有的行。查看 keyvalue获得更多关于HBase内部如何存储数据的信息。

## 30. Joins
HBase是否支持join是一个常见的问题，答案是没有，至少没办法像RDBMS那样支持（例如等价式join或者外部join）。正如本章节所阐述的，HBase中读取数据的操作是Get和Scan。

然而，这不意味着等价式join功能没办法在你的应用中实现，但是你必须自己实现。两种主要策略是将数据非结构化地写到HBase中，或者查找表然后在应用中或者MapReduce代码中实现join操作（正如RDBMS所演示的，将根据表的大小会有几种不同的策略，例如嵌套使循环和hash-join）。哪个是最好的方法？这将依赖于你想做什么，没有一种方案能够应对各种情况。

## 31. ACID
查看 ACID Semantics. Lars Hofhansl也写了一份报告 ACID in HBase.
