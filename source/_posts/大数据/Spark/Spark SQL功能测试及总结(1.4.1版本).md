---
title: Spark SQL功能测试及总结(1.4.1版本)
date: 2016-06-11 16:48:05
tags: [Spark]
categories: [大数据]
---
### 背景
spark sql在项目中使用越来越多，spark sql都支持哪些功能？官网没有明确说明，只能在class SqlParserd代码中看到一些Keyword，所以准备测试下spark对常用sql的支持情况。

### 一、测试总结
````
1、结果默认正向排序
2、distinct 去重：大小写敏感，可作用于其后多个字段
3、limit 限制返回结果条数：前n条
4、排序 order by：作用于其后多列(可分别指定排序方式)；排序字段可为非选择列；默认正续；不支持列位置排序(不抛异常，按第一列正续)
5、支持的where子句操作符：=,<>,!=,<,<=,>,>=,between,is null,is not null
6、组合where子句：and、or、in、not，and 优先级高于 or；not 可修饰in；为表达方便，or与in建议优先选择in
7、模糊查找like通配符：%，- ，通配符对数据类型也起作用
8、支持表和字段别名
9、不支持字段拼接
10、支持算术计算：+,-,*,/,%
11、字符串函数：lower、upper，没去空格函数
12、数值函数 ：abs、sqrt，没三角函数
13、聚集函数：avg、count、sum、max、min，avg忽略为空的行，作用于字符串返回null；count 不忽略空行；max、min也可以作用于字符串列，分别返回第一行最后一行；avg，max，min 不支持去重后求值
14、支持分组 group by及分组过滤having
15、支持表内联结、左外联结、右外联结、全联结
16、组合union：union结果去重，union all不去重(功能同unionAll函数)
17、不支持子查询(2.0版本支持：https://issues.apache.org/jira/browse/SPARK-4226)
````

### 二、测试数据
表1：Products
````
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
|   BR01|  BRS01|  8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01| 12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01| 18 inch teddy bear|     11.99|  18 inch teddy bear|
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
|  RYL01|  FNG01|          King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|         Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|         Queen doll|       0.0|12 inch queen dol...|
+-------+-------+-------------------+----------+--------------------+
````
表2：Vendors
````
+-------+---------------+---------------+----------+----------+--------+------------+
|vend_id|      vend_name|   vend_address| vend_city|vend_state|vend_zip|vend_country|
+-------+---------------+---------------+----------+----------+--------+------------+
|  BRS01|     Bears R Us|123 Main Street| Bear Town|        MI|   44444|         USA|
|  BRE02|  Bear Emporium|500 Park Street|   Anytown|        OH|   44333|         USA|
|  DLL01|Doll House Inc.|555 High Street|Dollsville|        CA|   99999|         USA|
|  FRB01|   Furball Inc.|1000 5th Avenue|  New York|        NY|   11111|         USA|
|  FNG01|  Fun and Games| 42 Galaxy Road|    London|      NULL| N16 6PS|     England|
|  JTS01| Jouets et ours|1 Rue Amusement|     Paris|      NULL|   45678|      France|
+-------+---------------+---------------+----------+----------+--------+------------+
````
### 三、测试sql及结果
1、数据检索
1）检索列
````
ssc.sql("select prod_name from Products").show
+-------------------+
|          prod_name|
+-------------------+
|  8 inch teddy bear|
| 12 inch teddy bear|
| 18 inch teddy bear|
|  Fish bean bag toy|
|  Bird bean bag toy|
|Rabbit bean bag toy|
|        Raggedy Ann|
|          King doll|
|         Queen doll|
|         Queen doll|
+-------------------+
ssc.sql("select prod_id, prod_name,prod_price from Products").show
+-------+-------------------+----------+
|prod_id|          prod_name|prod_price|
+-------+-------------------+----------+
|   BR01|  8 inch teddy bear|      5.99|
|   BR02| 12 inch teddy bear|      8.99|
|   BR03| 18 inch teddy bear|     11.99|
| BNBG01|  Fish bean bag toy|      3.49|
| BNBG02|  Bird bean bag toy|      3.49|
| BNBG03|Rabbit bean bag toy|      3.49|
| RGAN01|        Raggedy Ann|      4.99|
|  RYL01|          King doll|      9.49|
|  RYL02|         Queen doll|      9.49|
|  ryl02|         Queen doll|       0.0|
+-------+-------------------+----------+
select * from Products
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
|   BR01|  BRS01|  8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01| 12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01| 18 inch teddy bear|     11.99|  18 inch teddy bear|
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
|  RYL01|  FNG01|          King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|         Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|         Queen doll|       0.0|12 inch queen dol...|
+-------+-------+-------------------+----------+--------------------+
````
说明：返回结果默认按第一列正向排序
2）检索不同的值 distinct
````
ssc.sql("select distinct vend_id from Products").show
+-------+
|vend_id|
+-------+
|  DLL01|
|  FNG01|
|  BRS01|
|  fng01|
+-------+
````
说明：大小写敏感；可作用于其后多列

3）限制结果条数 limit
````
ssc.sql("select prod_name from Products limit 5")
+------------------+
|         prod_name|
+------------------+
| 8 inch teddy bear|
|12 inch teddy bear|
|18 inch teddy bear|
| Fish bean bag toy|
| Bird bean bag toy|
+------------------+
````
说明：取前n条
4）数据排序 order by
````
ssc.sql("select prod_id, prod_name from Products order by prod_id desc,prod_price").show
+-------+-------------------+
|prod_id|          prod_name|
+-------+-------------------+
|  ryl02|         Queen doll|
|  RYL02|         Queen doll|
|  RYL01|          King doll|
| RGAN01|        Raggedy Ann|
|   BR03| 18 inch teddy bear|
|   BR02| 12 inch teddy bear|
|   BR01|  8 inch teddy bear|
| BNBG03|Rabbit bean bag toy|
| BNBG02|  Bird bean bag toy|
| BNBG01|  Fish bean bag toy|
+-------+-------------------+
ssc.sql("select prod_id, prod_name from Products order by 0 desc,22,prod_price").show
+-------+-------------------+
|prod_id|          prod_name|
+-------+-------------------+
|   BR01|  8 inch teddy bear|
|   BR02| 12 inch teddy bear|
|   BR03| 18 inch teddy bear|
| BNBG01|  Fish bean bag toy|
| BNBG02|  Bird bean bag toy|
| BNBG03|Rabbit bean bag toy|
| RGAN01|        Raggedy Ann|
|  RYL01|          King doll|
|  RYL02|         Queen doll|
|  ryl02|         Queen doll|
+-------+-------------------+
````
说明：order by要作为select的最后一个子句；作用于其后多列(可分别指定排序方式)；排序字段可为非选择列；默认正续；不支持列位置排序(不抛异常，按第一列正续)
二、数据过滤
1）where 子句操作符：=,<>,!=,<,<=,>,>=,between,is null,is not null；
````
select * from Products where prod_price = 3.49
+-------+-------+-------------------+----------+-------------------+
|prod_id|vend_id|          prod_name|prod_price|          prod_desc|
+-------+-------+-------------------+----------+-------------------+
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|  Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|  Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49|Rabbit bean bag toy|
+-------+-------+-------------------+----------+-------------------+
ssc.sql("select * from Products where prod_price != 3.49").show
+-------+-------+------------------+----------+--------------------+
|prod_id|vend_id|         prod_name|prod_price|           prod_desc|
+-------+-------+------------------+----------+--------------------+
|   BR01|  BRS01| 8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01|12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01|18 inch teddy bear|     11.99|  18 inch teddy bear|
| RGAN01|  DLL01|       Raggedy Ann|      4.99|18 inch Raggedy A...|
|  RYL01|  FNG01|         King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|        Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|        Queen doll|       0.0|12 inch queen dol...|
+-------+-------+------------------+----------+--------------------+
ssc.sql("select * from Products where prod_price <= 5").show
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
|  ryl02|  fng01|         Queen doll|       0.0|12 inch queen dol...|
+-------+-------+-------------------+----------+--------------------+
ssc.sql("select * from Products where vend_id != 'DLL01'").show
+-------+-------+------------------+----------+--------------------+
|prod_id|vend_id|         prod_name|prod_price|           prod_desc|
+-------+-------+------------------+----------+--------------------+
|   BR01|  BRS01| 8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01|12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01|18 inch teddy bear|     11.99|  18 inch teddy bear|
|  RYL01|  FNG01|         King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|        Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|        Queen doll|       0.0|12 inch queen dol...|
+-------+-------+------------------+----------+--------------------+
ssc.sql("select * from Products where prod_price between 5 and 10").show
+-------+-------+------------------+----------+--------------------+
|prod_id|vend_id|         prod_name|prod_price|           prod_desc|
+-------+-------+------------------+----------+--------------------+
|   BR01|  BRS01| 8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01|12 inch teddy bear|      8.99|  12 inch teddy bear|
|  RYL01|  FNG01|         King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|        Queen doll|      9.49|12 inch queen dol...|
+-------+-------+------------------+----------+--------------------+
ssc.sql("select * from Products where prod_price is null ").show
+-------+-------+---------+----------+---------+
|prod_id|vend_id|prod_name|prod_price|prod_desc|
+-------+-------+---------+----------+---------+
+-------+-------+---------+----------+---------+
ssc.sql("select * from Products where prod_price is not null ").show
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
|   BR01|  BRS01|  8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01| 12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01| 18 inch teddy bear|     11.99|  18 inch teddy bear|
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
|  RYL01|  FNG01|          King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|         Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|         Queen doll|       0.0|12 inch queen dol...|
+-------+-------+-------------------+----------+--------------------+
````
2）组合where子句
````
ssc.sql("select * from Products where vend_id = 'DLL01' or vend_id = 'BRS01'").show
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
|   BR01|  BRS01|  8 inch teddy bear|      5.99|   8 inch teddy bear|
|   BR02|  BRS01| 12 inch teddy bear|      8.99|  12 inch teddy bear|
|   BR03|  BRS01| 18 inch teddy bear|     11.99|  18 inch teddy bear|
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
+-------+-------+-------------------+----------+--------------------+
ssc.sql("select * from Products where vend_id = 'DLL01' or vend_id = 'BRS01' and prod_price > 10").show
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
|   BR03|  BRS01| 18 inch teddy bear|     11.99|  18 inch teddy bear|
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
+-------+-------+-------------------+----------+--------------------+
ssc.sql("select * from Products where (vend_id = 'DLL01' or vend_id = 'BRS01') and prod_price > 10").show
+-------+-------+------------------+----------+------------------+
|prod_id|vend_id|         prod_name|prod_price|         prod_desc|
+-------+-------+------------------+----------+------------------+
|   BR03|  BRS01|18 inch teddy bear|     11.99|18 inch teddy bear|
+-------+-------+------------------+----------+------------------+
ssc.sql("select * from Products where vend_id in ('DLL01','BRS01') and prod_price > 10").show
+-------+-------+------------------+----------+------------------+
|prod_id|vend_id|         prod_name|prod_price|         prod_desc|
+-------+-------+------------------+----------+------------------+
|   BR03|  BRS01|18 inch teddy bear|     11.99|18 inch teddy bear|
+-------+-------+------------------+----------+------------------+
ssc.sql("select * from Products where vend_id not in ('DLL01','BRS01')").show
+-------+-------+----------+----------+--------------------+
|prod_id|vend_id| prod_name|prod_price|           prod_desc|
+-------+-------+----------+----------+--------------------+
|  RYL01|  FNG01| King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|Queen doll|       0.0|12 inch queen dol...|
+-------+-------+----------+----------+--------------------+
````
总结：and 优先级高于 or；not 可修饰in；为表达方便，or与in建议优先选择in

3）通配符
````
ssc.sql("select * from Products where prod_name like '%doll%'").show
+-------+-------+----------+----------+--------------------+
|prod_id|vend_id| prod_name|prod_price|           prod_desc|
+-------+-------+----------+----------+--------------------+
|  RYL01|  FNG01| King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|Queen doll|      9.49|12 inch queen dol...|
|  ryl02|  fng01|Queen doll|       0.0|12 inch queen dol...|
+-------+-------+----------+----------+--------------------+
ssc.sql("select * from Products where prod_price like '%4%'").show
+-------+-------+-------------------+----------+--------------------+
|prod_id|vend_id|          prod_name|prod_price|           prod_desc|
+-------+-------+-------------------+----------+--------------------+
| BNBG01|  DLL01|  Fish bean bag toy|      3.49|   Fish bean bag toy|
| BNBG02|  DLL01|  Bird bean bag toy|      3.49|   Bird bean bag toy|
| BNBG03|  DLL01|Rabbit bean bag toy|      3.49| Rabbit bean bag toy|
| RGAN01|  DLL01|        Raggedy Ann|      4.99|18 inch Raggedy A...|
|  RYL01|  FNG01|          King doll|      9.49|12 inch king doll...|
|  RYL02|  FNG01|         Queen doll|      9.49|12 inch queen dol...|
+-------+-------+-------------------+----------+--------------------+
ssc.sql("select * from Products where prod_name like '__ inch teddy bear'"
+-------+-------+------------------+----------+------------------+
|prod_id|vend_id|         prod_name|prod_price|         prod_desc|
+-------+-------+------------------+----------+------------------+
|   BR02|  BRS01|12 inch teddy bear|      8.99|12 inch teddy bear|
|   BR03|  BRS01|18 inch teddy bear|     11.99|18 inch teddy bear|
+-------+-------+------------------+----------+------------------+
````
4）不支持字段拼接
````
ssc.sql("select prod_name  + '-' + prod_id from Products").show
+----+
|  c0|
+----+
|null|
|null|
|null|
|null|
|null|
|null|
|null|
|null|
|null|
|null|
+----+
````

5）字段算数运算 (+,-,*,/,%)
````
ssc.sql("select prod_name ,prod_price,prod_price*2 from Products").show
+-------------------+----------+-----+
|          prod_name|prod_price|   c2|
+-------------------+----------+-----+
|  8 inch teddy bear|      5.99|11.98|
| 12 inch teddy bear|      8.99|17.98|
| 18 inch teddy bear|     11.99|23.98|
|  Fish bean bag toy|      3.49| 6.98|
|  Bird bean bag toy|      3.49| 6.98|
|Rabbit bean bag toy|      3.49| 6.98|
|        Raggedy Ann|      4.99| 9.98|
|          King doll|      9.49|18.98|
|         Queen doll|      9.49|18.98|
|         Queen doll|       0.0|  0.0|
+-------------------+----------+-----+
````
6）字符串函数（不支持去空格）
````
ssc.sql("select lower(prod_name),upper(prod_name),prod_price from Products").show
+-------------------+-------------------+----------+
|                 c0|                 c1|prod_price|
+-------------------+-------------------+----------+
|  8 inch teddy bear|  8 INCH TEDDY BEAR|      5.99|
| 12 inch teddy bear| 12 INCH TEDDY BEAR|      8.99|
| 18 inch teddy bear| 18 INCH TEDDY BEAR|     11.99|
|  fish bean bag toy|  FISH BEAN BAG TOY|      3.49|
|  bird bean bag toy|  BIRD BEAN BAG TOY|      3.49|
|rabbit bean bag toy|RABBIT BEAN BAG TOY|      3.49|
|        raggedy ann|        RAGGEDY ANN|      4.99|
|          king doll|          KING DOLL|      9.49|
|         queen doll|         QUEEN DOLL|      9.49|
|         queen doll|         QUEEN DOLL|       0.0|
+-------------------+-------------------+----------+
````
7）数值函数
````
ssc.sql("select prod_price, abs(prod_price), sqrt(prod_price) from Products").show
+----------+-----+------------------+
|prod_price|   c1|                c2|
+----------+-----+------------------+
|      5.99| 5.99|2.4474476501040834|
|      8.99| 8.99|  2.99833287011299|
|     11.99|11.99| 3.462657938636157|
|      3.49| 3.49|1.8681541692269406|
|      3.49| 3.49|1.8681541692269406|
|      3.49| 3.49|1.8681541692269406|
|      4.99| 4.99| 2.233830790368868|
|      9.49| 9.49|3.0805843601498726|
|      9.49| 9.49|3.0805843601498726|
|       0.0|  0.0|               0.0|
+----------+-----+------------------+
````
8）聚集函数
````
ssc.sql("select avg(prod_price) avg_price from Products").show
+-----------------+
|        avg_price|
+-----------------+
|6.141000000000001|
+-----------------+
ssc.sql("select count(prod_price) c from Products").show
+--+
| c|
+--+
|10|
+--+
````
说明：avg忽略为空的行，作用于字符串返回null；count 不忽略空行
````
ssc.sql("select max(prod_price) m from Products").show
+-----+
|    m|
+-----+
|11.99|
+-----+
````
说明：同min也可以作用于字符串列，分别返回第一行最后一行
````
ssc.sql("select sum(prod_name) s from Products").show
+---+
|  s|
+---+
|0.0|
+---+
ssc.sql("select sum(distinct prod_price),count(distinct prod_price)from Products").show
+------------------+--+
|                c0|c1|
+------------------+--+
|44.940000000000005| 7|
+------------------+--+
````
说明：avg，max，min 不支持去重后求值
9）分组 group by
````
ssc.sql("select vend_id ,count(*),max(prod_price),avg(prod_price) from Products group by vend_id").show
+-------+--+-----+-----+
|vend_id|c1|   c2|   c3|
+-------+--+-----+-----+
|  DLL01| 4| 4.99|3.865|
|  FNG01| 2| 9.49| 9.49|
|  BRS01| 3|11.99| 8.99|
|  fng01| 1|  0.0|  0.0|
+-------+--+-----+-----+
ssc.sql("select vend_id ,count(*) c,max(prod_price) m,avg(prod_price)  from Products where prod_price > 0 group by vend_id having m > 5 and c > 2").show
+-------+-+-----+----+
|vend_id|c|    m|  c3|
+-------+-+-----+----+
|  BRS01|3|11.99|8.99|
+-------+-+-----+----+
ssc.sql("select vend_id ,count(*) c,max(prod_price) m,avg(prod_price)  from Products where prod_price > 0 group by vend_id having m > 5 order by c3").show
+-------+-+-----+----+
|vend_id|c|    m|  c3|
+-------+-+-----+----+
|  BRS01|3|11.99|8.99|
|  FNG01|2| 9.49|9.49|
+-------+-+-----+----+
````

10）过滤分组
````
ssc.sql("select vend_id from Products group by vend_id having avg(prod_price) > 3").show
+-------+
|vend_id|
+-------+
|  DLL01|
|  FNG01|
|  BRS01|
+-------+
````
11）表联结
````
ssc.sql("select vend_name,prod_name,prod_price from Vendors v,Products p where v.vend_id = p.vend_id ").show
ssc.sql("select vend_name,prod_name,prod_price from Vendors v inner join Products p on v.vend_id = p.vend_id ").show
+---------------+-------------------+----------+
|      vend_name|          prod_name|prod_price|
+---------------+-------------------+----------+
|Doll House Inc.|  Fish bean bag toy|      3.49|
|Doll House Inc.|  Bird bean bag toy|      3.49|
|Doll House Inc.|Rabbit bean bag toy|      3.49|
|Doll House Inc.|        Raggedy Ann|      4.99|
|  Fun and Games|          King doll|      9.49|
|  Fun and Games|         Queen doll|      9.49|
|     Bears R Us|  8 inch teddy bear|      5.99|
|     Bears R Us| 12 inch teddy bear|      8.99|
|     Bears R Us| 18 inch teddy bear|     11.99|
+---------------+-------------------+----------+
ssc.sql("select vend_name,prod_name,prod_price from Vendors v left join Products p on v.vend_id = p.vend_id ").show
+---------------+-------------------+----------+
|      vend_name|          prod_name|prod_price|
+---------------+-------------------+----------+
|  Bear Emporium|               null|      null|
|   Furball Inc.|               null|      null|
| Jouets et ours|               null|      null|
|Doll House Inc.|  Fish bean bag toy|      3.49|
|Doll House Inc.|  Bird bean bag toy|      3.49|
|Doll House Inc.|Rabbit bean bag toy|      3.49|
|Doll House Inc.|        Raggedy Ann|      4.99|
|  Fun and Games|          King doll|      9.49|
|  Fun and Games|         Queen doll|      9.49|
|     Bears R Us|  8 inch teddy bear|      5.99|
|     Bears R Us| 12 inch teddy bear|      8.99|
|     Bears R Us| 18 inch teddy bear|     11.99|
+---------------+-------------------+----------+
ssc.sql("select vend_name,prod_name,prod_price from Vendors v right outer join Products p on v.vend_id = p.vend_id ").show
+---------------+-------------------+----------+
|      vend_name|          prod_name|prod_price|
+---------------+-------------------+----------+
|Doll House Inc.|  Fish bean bag toy|      3.49|
|Doll House Inc.|  Bird bean bag toy|      3.49|
|Doll House Inc.|Rabbit bean bag toy|      3.49|
|Doll House Inc.|        Raggedy Ann|      4.99|
|  Fun and Games|          King doll|      9.49|
|  Fun and Games|         Queen doll|      9.49|
|     Bears R Us|  8 inch teddy bear|      5.99|
|     Bears R Us| 12 inch teddy bear|      8.99|
|     Bears R Us| 18 inch teddy bear|     11.99|
|           null|         Queen doll|       0.0|
+---------------+-------------------+----------+
ssc.sql("select vend_name,prod_name,prod_price from Vendors v full outer join Products p on v.vend_id = p.vend_id ").show
+---------------+-------------------+----------+
|      vend_name|          prod_name|prod_price|
+---------------+-------------------+----------+
|   Furball Inc.|               null|      null|
|  Bear Emporium|               null|      null|
| Jouets et ours|               null|      null|
|Doll House Inc.|  Fish bean bag toy|      3.49|
|Doll House Inc.|  Bird bean bag toy|      3.49|
|Doll House Inc.|Rabbit bean bag toy|      3.49|
|Doll House Inc.|        Raggedy Ann|      4.99|
|  Fun and Games|          King doll|      9.49|
|  Fun and Games|         Queen doll|      9.49|
|     Bears R Us|  8 inch teddy bear|      5.99|
|     Bears R Us| 12 inch teddy bear|      8.99|
|     Bears R Us| 18 inch teddy bear|     11.99|
|           null|         Queen doll|       0.0|
+---------------+-------------------+----------+
ssc.sql("select vend_name ,count(prod_id) c  from Products p left outer join  Vendors v on v.vend_id = p.vend_id group by vend_name").show
+---------------+-+
|      vend_name|c|
+---------------+-+
|  Fun and Games|2|
|Doll House Inc.|4|
|     Bears R Us|3|
|           null|1|
+---------------+-+
````
12）组合查询
````
ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI'").unionAll(ssc.sql("select * from Vendors v where v.vend_state in ('MI','OH')")).show
+-------+--------------+---------------+---------+----------+--------+------------+
|vend_id|     vend_name|   vend_address|vend_city|vend_state|vend_zip|vend_country|
+-------+--------------+---------------+---------+----------+--------+------------+
|  BRS01|    Bears R Us|123 Main Street|Bear Town|        MI|   44444|         USA|
|  JTS01|Jouets et ours|1 Rue Amusement|    Paris|      NULL|   45678|      France|
|  BRS01|    Bears R Us|123 Main Street|Bear Town|        MI|   44444|         USA|
|  BRE02| Bear Emporium|500 Park Street|  Anytown|        OH|   44333|         USA|
+-------+--------------+---------------+---------+----------+--------+------------+
ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI' union select * from Vendors v where v.vend_state in ('MI','OH')").show
+-------+--------------+---------------+---------+----------+--------+------------+
|vend_id|     vend_name|   vend_address|vend_city|vend_state|vend_zip|vend_country|
+-------+--------------+---------------+---------+----------+--------+------------+
|  BRS01|    Bears R Us|123 Main Street|Bear Town|        MI|   44444|         USA|
|  JTS01|Jouets et ours|1 Rue Amusement|    Paris|      NULL|   45678|      France|
|  BRE02| Bear Emporium|500 Park Street|  Anytown|        OH|   44333|         USA|
+-------+--------------+---------------+---------+----------+--------+------------+
ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI' union all select * from Vendors v where v.vend_state in ('MI','OH')").show
+-------+--------------+---------------+---------+----------+--------+------------+
|vend_id|     vend_name|   vend_address|vend_city|vend_state|vend_zip|vend_country|
+-------+--------------+---------------+---------+----------+--------+------------+
|  BRS01|    Bears R Us|123 Main Street|Bear Town|        MI|   44444|         USA|
|  JTS01|Jouets et ours|1 Rue Amusement|    Paris|      NULL|   45678|      France|
|  BRS01|    Bears R Us|123 Main Street|Bear Town|        MI|   44444|         USA|
|  BRE02| Bear Emporium|500 Park Street|  Anytown|        OH|   44333|         USA|
+-------+--------------+---------------+---------+----------+--------+------------+
````
说明：union 排重
四、代码
````
import org.apache.spark.sql.types._
import org.apache.spark.sql.{Row, SQLContext}
import org.apache.spark.{SparkConf, SparkContext}

/**
 * Created by xuemin.zhang on 16/6/12.
 */
object SqlTestOnParquet {
  def main(args: Array[String]) {
    val conf = new SparkConf().setMaster("local").setAppName("test")
    val sc = new SparkContext(conf)
    val ssc = new SQLContext(sc)

    val products_data = Array(
      "BR01,BRS01,8 inch teddy bear,5.99,8 inch teddy bear,comes with cap and jacket",
      "BR02,BRS01,12 inch teddy bear,8.99,12 inch teddy bear,comes with cap and jacket",
      "BR03,BRS01,18 inch teddy bear,11.99,18 inch teddy bear,comes with cap and jacket",
      "BNBG01,DLL01,Fish bean bag toy,3.49,Fish bean bag toy,complete with bean bag worms with which to feed it",
      "BNBG02,DLL01,Bird bean bag toy,3.49,Bird bean bag toy,eggs are not included",
      "BNBG03,DLL01,Rabbit bean bag toy,3.49,Rabbit bean bag toy,comes with bean bag carrots",
      "RGAN01,DLL01,Raggedy Ann,4.990,18 inch Raggedy Ann doll",
      "RYL01,FNG01,King doll,9.49,12 inch king doll with royal garments and crown",
      "RYL02,FNG01,Queen doll,9.49,12 inch queen doll with royal garments and crown",
      "ryl02,fng01,Queen doll,0,12 inch queen doll with royal garments and crown"
    )

    val productsRdd = sc.parallelize(products_data).map(_.split(",")).map(p => Row(p(0), p(1), p(2), p(3).toDouble, p(4)))
    val productsFields = Array(
      StructField("prod_id",StringType,true),
      StructField("vend_id",StringType,true),
      StructField("prod_name",StringType,true),
      StructField("prod_price",DoubleType,true),
      //StructField("prod_price",StringType,true),
      //StructField("prod_price",IntegerType,true),
      StructField("prod_desc",StringType,true)//介绍
    )
    val productsSchema = StructType(productsFields)
    val productsDf = ssc.createDataFrame(productsRdd,productsSchema)
    productsDf.registerTempTable("Products")

    productsDf.printSchema()
    productsDf.show()

    ssc.sql("select prod_name from Products").show
    ssc.sql("select prod_id, prod_name,prod_price from Products").show
    ssc.sql("select * from Products").show
    ssc.sql("select vend_id from Products").show
    ssc.sql("select distinct vend_id from Products").show //大小写敏感；作用于其后多列
    ssc.sql("select prod_name from Products limit 5").show
    ssc.sql("select prod_id, prod_name from Products order by prod_id desc,prod_price").show //select 语句的最后一个子句 ；作用于其后多列（可分别指定排序方式）；排序字段可为非显示列
    ssc.sql("select prod_id, prod_name from Products order by 22 desc,prod_price").show //select 语句的最后一个子句 ；作用于其后多列（可分别指定排序方式）；排序字段可为非显示列

    ssc.sql("select * from Products where prod_price = 3.49").show
    ssc.sql("select * from Products where prod_price !< 3.49").show
    ssc.sql("select * from Products where prod_price <= 5").show
    ssc.sql("select * from Products where vend_id != 'DLL01'").show //字符串类型比较，使用单引号
    ssc.sql("select * from Products where prod_price between 5 and 10").show
    ssc.sql("select * from Products where prod_price is null ").show
    ssc.sql("select * from Products where prod_price is not null ").show

    ssc.sql("select * from Products where vend_id = 'DLL01' or vend_id = 'BRS01'").show
    ssc.sql("select * from Products where vend_id = 'DLL01' or vend_id = 'BRS01' and prod_price > 10").show
    ssc.sql("select * from Products where (vend_id = 'DLL01' or vend_id = 'BRS01') and prod_price > 10").show
    ssc.sql("select * from Products where vend_id in ('DLL01','BRS01')").show //建议优先使用in
    ssc.sql("select * from Products where vend_id in ('DLL01','BRS01') and prod_price > 10").show
    ssc.sql("select * from Products where vend_id not in ('DLL01','BRS01')").show //建议优先使用in


    ssc.sql("select * from Products where prod_name like '%doll%'").show
    ssc.sql("select * from Products where prod_price like '%4%'").show //不仅仅字符能用通配符
    ssc.sql("select * from Products where prod_name like '__ inch teddy bear'").show
    ssc.sql("select * from Products where prod_name like '% inch teddy bear'").show
    ////ssc.sql("select * from Products where prod_name like '1[2,8] inch teddy bear'").show //不支持[]

    ssc.sql("select prod_name  + '-' + prod_id from Products").show // 不支持列拼接
    ssc.sql("select prod_name ,prod_price,prod_price*2 from Products").show // 执行运算 + - * /
    ssc.sql("select prod_name ,prod_price,prod_price%2 from Products").show // 执行运算 + - * /


    ssc.sql("select lower(prod_name),upper(prod_name),prod_price from Products").show //
    ssc.sql("select trim(prod_name) from Products").show // 不支持去空格
    ssc.sql("select prod_price, abs(prod_price), sqrt(prod_price) from Products").show

    ssc.sql("select avg(prod_price) avg_price from Products").show //avg忽略为空的行 ，作用于字符串返回null
    ssc.sql("select count(prod_price) c from Products").show //不忽略为空的行
    ssc.sql("select max(prod_price) m from Products").show // 同min也可以作用于字符串列，分别返回第一行最后一行
    ssc.sql("select sum(prod_name) s from Products").show //作用于字符串返回0.0
    ssc.sql("select sum(distinct prod_price),count(distinct prod_price)from Products").show //avg，max，min 不支持去重后求值

    ssc.sql("select vend_id ,count(*),max(prod_price),avg(prod_price) from Products group by vend_id").show
    ssc.sql("select vend_id ,count(*) c,max(prod_price) m,avg(prod_price)  from Products where prod_price > 0 group by vend_id having m > 5 and c > 2").show
    ssc.sql("select vend_id ,count(*) c,max(prod_price) m,avg(prod_price)  from Products where prod_price > 0 group by vend_id having m > 5 order by c3").show

    ssc.sql("select prod_name p from Products").show // 别名
    ssc.sql("select prod_name,upper(prod_name),prod_price from Products").show //

    val vendorsData = Array(
      "BRS01,Bears R Us,123 Main Street,Bear Town,MI,44444,USA",
      "BRE02,Bear Emporium,500 Park Street,Anytown,OH,44333,USA",
      "DLL01,Doll House Inc.,555 High Street,Dollsville,CA,99999,USA",
      "FRB01,Furball Inc.,1000 5th Avenue,New York,NY,11111,USA",
      "FNG01,Fun and Games,42 Galaxy Road,London,NULL,N16 6PS,England",
      "JTS01,Jouets et ours,1 Rue Amusement,Paris,NULL,45678,France"
    )
    val vendorsRdd = sc.parallelize(vendorsData).map(_.split(",")).map(p => Row(p(0), p(1), p(2), p(3), p(4), p(5), p(6)))
    val vendorsFields = Array(
      StructField("vend_id",StringType,false),
      StructField("vend_name",StringType,false),
      StructField("vend_address",StringType,true),
      StructField("vend_city",StringType,true),
      StructField("vend_state",StringType,true),
      StructField("vend_zip",StringType,true),
      StructField("vend_country",StringType,true))
    val vendorsSchema = StructType(vendorsFields)
    val vendorsDf = ssc.createDataFrame(vendorsRdd,vendorsSchema)
    vendorsDf.registerTempTable("Vendors")

    vendorsDf.printSchema()
    vendorsDf.show()
    ssc.sql("select vend_id from Products group by vend_id having avg(prod_price) > 3").show
    // ssc.sql("select * from Vendors where vend_id in (select vend_id from Products group by vend_id having avg(prod_price) > 3)").show //subquery在2.0支持https://issues.apache.org/jira/browse/SPARK-4226

    ssc.sql("select vend_name,prod_name,prod_price from Vendors v,Products p where v.vend_id = p.vend_id ").show
    ssc.sql("select vend_name,prod_name,prod_price from Vendors v inner join Products p on v.vend_id = p.vend_id ").show

    ssc.sql("select vend_name,prod_name,prod_price from Vendors v left join Products p on v.vend_id = p.vend_id ").show
    ssc.sql("select vend_name,prod_name,prod_price from Vendors v right outer join Products p on v.vend_id = p.vend_id ").show
    ssc.sql("select vend_name,prod_name,prod_price from Vendors v full outer join Products p on v.vend_id = p.vend_id ").show

    ssc.sql("select vend_name ,count(prod_id) c  from Products p left outer join  Vendors v on v.vend_id = p.vend_id group by vend_name").show

    ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI'").unionAll(ssc.sql("select * from Vendors v where v.vend_state in ('MI','OH')")).show
    ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI' union select * from Vendors v where v.vend_state in ('MI','OH')").show
    ssc.sql("select * from Vendors v where vend_id = 'JTS01' or vend_state= 'MI' union all select * from Vendors v where v.vend_state in ('MI','OH')").show

    ssc.sql("INSERT into table Vendors select * from Vendors").show

  }
}
````
