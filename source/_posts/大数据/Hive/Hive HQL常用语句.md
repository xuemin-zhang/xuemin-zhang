---
title: Excle时间戳(绝对秒、毫秒)与日期格式互转
date: 2016-03-11 16:48:05
copyright:
tags: [Hive]
categories: [大数据]
---



show databases; # 查看某个数据库
create database 库名;
use 数据库;      # 进入某个数据库
show tables;    # 展示所有表
desc 表名;            # 显示表结构

create table t1(name string ,age int);


load data local inpath '/home/bigdata' into table hive.dep;

insert overwrite table tab_name select * from tab_name2;
create table tab_name as select * from t_name2;

insert overwrite local directory '/home/hadoop/test' select *from t_name;
insert overwrite directory '/aaa/bbb/' select *from t_p;  将查询结果保存到指定的文件目录（可以是本地，也可以HDFS）



alter table old_name  rename to new_name;


alter table tab_name change column key key_1 int comment 'h' after value; 修改某一个表的某一列


alter table tab_name add columns(value1 string,value2 string); 增加某一列


alter table tab_name replace columns(values string,value11 string); 替换表中的某一个列
