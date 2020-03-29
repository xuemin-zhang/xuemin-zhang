---
title: Thrift二进制序列化TCompactProtocol与TBinaryProtocol的原理和区别
date: 2016-09-30 16:48:05
tags: [Thrift]
categories: [编程基础]
---
#### 背景
Thrift二进制序列化协议中,默认为TBinaryProtocol,关于TCompactProtocol的说明,为高效密集型的二进制序列化(varint).那么TCompactProtocol相对于TBinaryProtocol是怎样做到高效密集的呢?TCompactProtocol是否一定比TBinaryProtocol高效?


#### 我们以比较常用的i32类型为例,来解释一下两种方式各自的原理:

##### TBinaryProtocol
处理i32整型数据类型时,定义的是4个字节的数组,32位的长度正好可以保存到这4个字节组当中.如果我们分别以n1~n32来表示第1位到第32位,那么这个数组的数据结构应该为以下结构:
````
i32out[0] {n1  ~ n8 }
i32out[1] {n9  ~ n16}
i32out[2] {n17 ~ n24}
i32out[3] {n25 ~ n32}
````
这样的实现很简单.
对于其它类型,比如i16,也是类似的原理,不过是以2个字节的数组保存,在此不再说明了.

##### TCompactProtocol
在处理i32整型数据类型时,与TBinaryProtocol完全不同,采用的是1~5个字节组来保存.依然以n1~n32来表示第1位到第32位,数据结构应该为以下结构:
````
i32out[0] {1 , 0 , 0 , 0 , n1 ~ n4}
i32out[1] {1 , n5 ~ n11}
i32out[2] {1 , n12 ~ n18}
i32out[3] {1 , n19  ~ n25}
i32out[4] {0 , n26  ~ n32}
````
这是一种极端情况,5个字节全部占满.



很显然,这样做比TBinaryProtocol复杂得多,而且还多了1个字节,并没有达到密集的目的.那是不是说明TBinaryProtocol更好?先不急着下结论,举个具体一点的例子来说明两种实现的区别.
假如我们需要序列化一个十进制数值'10',那么它的二进制表示方式应该为'1010',只用了4位,但i32会在前面自动补0,则是'00000000000000000000000000001010',那么如果使用TBinaryProtocol方式来保存,则应该为以下结构:
````
i32out[0] {0 , 0 , 0 , 0 , 0 , 0 , 0 , 0}
i32out[1] {0 , 0 , 0 , 0 , 0 , 0 , 0 , 0}
i32out[2] {0 , 0 , 0 , 0 , 0 , 0 , 0 , 0}
i32out[3] {0 , 0 , 0 , 0 , 1 , 0 , 1 , 0}
````
这样有什么问题呢?那就是大量补足的0占用了宝贵空间.
接着我们再来看看TCompactProtocol会怎样保存'10'这个数值:
````
i32out[0] {0 , 0 , 0 , 0 , 1 , 0 , 1 , 0}
````
TCompactProtocol只用了1个字节,而TBinaryProtocol依然用了4个字节.这就是为什么说TCompactProtocol高效密集型的二进制序列化的原因.

#### 接着我们来分析一下TCompactProtocol的保存规则.
TCompactProtocol每个字节的第1位是状态位,第2位到第8位保存具体的数据.这有别于TBinaryProtocol的1到8位全部保存具体数据.这也是为什么极端情况下TCompactProtocol比TBinaryProtocol多占1个字节的原因.
TCompactProtocol的字节中第1位状态位的意思是标记此字节后是否还有数据.1为有数据,0为没有数据.
为了更容易理解,我们再举一个例子.
用TCompactProtocol来序列化十进制数值'300',二进制应该为'100101100',用TCompactProtocol方式来保存则为如下结构:
````
i32out[0] {1 , 0 , 0 , 0 , 0 , 0 , 1 , 0}
i32out[1] {0 , 0 , 1 , 0 , 1 , 1 , 0 , 0}
````
这里将'100101100'拆分为了2部分,'10'和'0101100',在i32out[0]中保存了'10',并在第1位记为'1'来表示后面还有,第7位和第8位保存'10',不足的几位(第2位到第6位)补0,所以i32out[0]为'10000010',在i32out[1]中保存了后面的'0101100',并在第1位记为'0'来表示后面没有了,则第1位为'0',所以i32out[1]为'00101100'.
TCompactProtocol以这样的原理来达到压缩的目的.

最后引用一张图来展示TCompactProtocol的i32数据('106903'数值序列化后)的结构:
![](https://static.oschina.net/uploads/space/2016/0926/194645_9dmf_766562.png)


#### 总结一下
TCompactProtocol在数据量较小的情况下比TBinaryProtocol序列化更省空间.
在i32数值或者string的长度大于'2097151'时(二进制21个'1'),与TBinaryProtocol方式占用的空间一样,都是4个字节.
在i32数值或者string的长度大于'268435455'时(二进制28个'1'),TCompactProtocol反而会比TBinaryProtocol多占用1个字节的空间.(其它数据类型依此类推,不做阐述了)


来源：https://my.oschina.net/venwyhk/blog/751790
