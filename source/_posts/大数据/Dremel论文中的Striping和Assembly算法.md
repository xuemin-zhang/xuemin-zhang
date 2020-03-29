---
title: Dremel论文中的Striping和Assembly算法
date: 2018-12-02 16:48:05
tags: [算法]
categories: [大数据]
---
为了理解Dremel论文中给出的案例，笔者觉得对definition level 和 repetition level这两个概念进行解释，对加强理解是有必要的，具体可以看Dremel那篇论文的图2和图3。


### 列式嵌套schema
论文中使用下面的schema举例：
```
message Document {
	required int64 DocId;
	optional group Links {
		repeated int64 Backward;
		repeated int64 Forward;
	}
	repeated group Name {
		repeated group Language {
			required string Code;
			optional string Country;
		}
		optional string Url;
	}
}

```

这个schema可以被看做叶子节点全是简单类型的树，一个叶子节点对应一个列，所以这个例子中有6个列：

```
DocId
Links.Backward
Links.Forward
Name.Language.Code
Name.Language.Country
Name.Url

```
因为其中一些字段是可重复字段，所以我们需要一些额外信息和数据一起存储，以便日后可以重组这些记录。这就是definition level 和 repetition level存在的意义了。


### 列拆解算法 Column striping algorithm

你可以把序列化存储一个记录看成是（深度优先）遍历一棵树。从记录的根节点开始，然后是第一个孩子节点直到到达一个叶子节点。孩子节点可以是一个不同的普通字段，也可以是一个不同的可重复字段。当到达一个叶子节点时，输出相应列的值和该列的最大definition level（意味着这个字段是被定义的）以及当前的repetition level。当我们到达一个空字段，


节点值和最后定义的在该节点之后的所有同类型的节点的定义级别都不会被写入（现有的重复级别也是如此，不被写入）。当回溯到树的上一个节点寻找另外的树枝继续遍历时，当前的重复级别为最后遇到的那个重复级别。



当到达一个叶子节点时，写出相应列里带有最大定义级别（这意味着这些字段是被定义的）的节点值和它现有的重复级别（我们开始访问的根节点的重复级别为0）。当我们到达一个空字段，节点值和最后定义的在该节点之后的所有同类型的节点的定义级别都不会被写入（现有的重复级别也是如此，不被写入）。当回溯到树的上一个节点寻找另外的树枝继续遍历时，当前的重复级别为最后遇到的那个重复级别。

当一个字段未被定义时，它的definition level会比该字段的definition level小。


### 记录重组

你想重构记录但只需要访问这些字段中的一个。重组记录就是把上文提到的树的遍历重做了一遍。选择你需要的字段并按如下方式重构：当一个节点的定义级别比最大的定义级别小，停止这一级的继续遍历；一旦访问到了叶子节点记录下它的重复级别同时开始从那个地方往下继续访问下一个值。


### definition level 和 repetition level

关于级别们的详细阐释可以在这个blog里找到：https://blog.twitter.com/2013/dremel-made-simple-with-parquet【不翻墙打不开】

理解这些例子很重要的一点是需要知道并不是树的每一个节点都需要定义级别或是重复级别。只有重复字段需要重复级别，只有那些选择性存在的字段会需要定义级别。因为这些级别值都很小所以它们可以被很高效地压缩。

必须存在的那些字段（requiredfields）永远都是被存在的并且不需要定义级别，非重复字段的字段不需要重复级别。

下图是论文中案例的结构模型注释，帮助大家更好地理解：

![](https://github.com/julienledem/redelm/raw/master/doc/dremel_paper/schema.png)

因为定义级别和重复级别被数据模型的深度所限制，所以它们都是很小的整数。用几个bit就可以存储。一个只有必须字段的扁平化的数据结构不需要任何多余的bits来存储这些永远会为0的级别。一个值为NULL并且定义了的可选字段需要一个额外的bit来存储0。NULL值本身是不需要被专门存储的因为定义级别已经包含了这一信息。

### 一步一步计算

#### R1
```
DocId: 10
Links
  Forward: 20
  Forward: 40
  Forward: 60
Name
  Language
    Code: 'en-us'
    Country: 'us'
  Language
    Code: 'en'
  Url: 'http://A'
Name
  Url: 'http://B'
Name
  Language
    Code: 'en-gb'
    Country: 'gb'

```
#### 输出

```

R = 0 (current repetition level)
DocId: 10, R:0, D:0
Links.Backward: NULL, R:0, D:1 (no value defined so D < 2)
Links.Forward: 20, R:0, D:2
R = 1 (we are repeating 'Links.Forward' of level 1)
Links.Forward: 40, R:1, D:2
R = 1 (we are repeating 'Links.Forward' again of level 1)
Links.Forward: 60, R:1, D:2
Back to the root level: R=0
Name.Language.Code: en-us, R:0, D:2
Name.Language.Country: us, R:0, D:3
R = 2 (we are repeating 'Name.Language' of level 2)
Name.Language.Code: en, R:2, D:2
Name.Language.Country: NULL, R:2, D:2 (no value defined so D < 3)
Name.Url: http://A, R:0, D:2
R = 1 (we are repeating 'Name' of level 1)
Name.Language.Code: NULL, R:1, D:1 (Only Name is defined so D = 1)
Name.Language.Country: NULL, R:1, D:1
Name.Url: http://B, R:1, D:2
R = 1 (we are repeating 'Name' again of level 1)
Name.Language.Code: en-gb, R:1, D:2
Name.Language.Country: gb, R:1, D:3
Name.Url: NULL, R:1, D:1

```

#### R2

```

DocId: 20
Links
  Backward: 10
  Backward: 30
  Forward:  80
Name
  Url: 'http://C'

```

#### 输出

```
DocId: 20, R:0, D:0
Links.Backward: 10, R:0, D:2
Links.Backward: 30, R:1, D:2
Links.Forward: 80, R:0, D:2
Name.Language.Code: NULL, R:0, D:1
Name.Language.Country: NULL, R:0, D:1
Name.Url: http://C, R:0, D:2

```



译：https://github.com/julienledem/redelm/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper



https://blog.csdn.net/macyang/article/details/8566105
