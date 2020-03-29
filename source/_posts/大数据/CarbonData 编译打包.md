---
title: CarbonData 编译打包
date: 2017-09-29 16:48:05
tags: [其他]
categories: [其他]
---
### 背景
CarbonData是什么？
官网说明：Apache CarbonData is an indexed columnar data format for fast analytics on big data platform, e.g. Apache Hadoop, Apache Spark, etc.
翻译过来即：Apache CarbonData 是一个含索引的列式数据格式，用于在如Apach Hadoop，Apach Spark等大数据平台上实现快速分析。

本想迅速试用一下，确发现官网没提供编译好的安装包，所以本篇文档记录下编译、打包的方法及过程。

### 一、基础条件
* Unix类操作系统(Linux，Mac OS X)
* Git
* Maven （建议使用3.3以上版本）
* Java 7 或者 8
* Thrift (0.9.3及以上）

### 二、Thrift 安装
其他基础组件都比较容易安装，这里分别记录下Linux、Mac OS X下Thrift的安装步骤

1、Linux系统环境(CentOS 6.5，root账号执行)

安装autoconf
````
  tar xvf autoconf-2.69.tar.gz
  cd autoconf-2.69
  ./configure --prefix=/usr
  make
  sudo make install
  cd ..
````
安装automake
````
wget http://ftp.gnu.org/gnu/automake/automake-1.14.tar.gz
tar xvf automake-1.14.tar.gz
cd automake-1.14
./configure --prefix=/usr
make
sudo make install
cd ..
````

安装bison
````
wget http://ftp.gnu.org/gnu/bison/bison-2.5.1.tar.gz
tar xvf bison-2.5.1.tar.gz
cd bison-2.5.1
./configure --prefix=/usr
make
sudo make install
cd ..
````

添加C++语言依赖库
````
yum -y install libevent-devel zlib-devel openssl-devel

````
安装boost(1.53以上)
````
wget http://sourceforge.net/projects/boost/files/boost/1.53.0/boost_1_53_0.tar.gz
tar xvf boost_1_53_0.tar.gz
cd boost_1_53_0
./bootstrap.sh
sudo ./b2 install
````

安装Thrift
````
cd thrift
git checkout 0.9.3
./bootstrap.sh
./configure --with-lua=no
make
sudo make install
````

2、Mac系统环境(OS X 10.11.4)
查看依赖 brew info thrift

安装依赖
````
brew install boost openssl libevent
````
使用安装包安装指定版本thrift

````
wget http://archive.apache.org/dist/thrift/0.9.3/thrift-0.9.3.tar.gz
tar -xcv thrift-0.9.3.tar.gz
cd thrift-0.9.3
./configure
make && make install
````
3、安装检验
````
thrift -version
````
### 三、下载编译CarbonData源码

1、下载源码
````
git clone https://github.com/apache/carbondata.git
````
2、编译打包

查看pom.xml内容发现，默认基于spark-2.1.0编译，上翻也有id为hadoop-2.2.0，hadoop-2.7.0相关profile，可以根据需求随意修改编译。

3、笔者使用：
````
mvn clean package -DskipTests -Pspark-1.6 -Dspark.version=1.6.3 -Phadoop-2.2.0 -Dhadoop.version=2.6.0
````
4、执行结果

如图表示编译成功，大概需要几分钟的时间。
成功后会在assembly/target/scala-2.10/目录下生成carbondata_2.10-1.1.1-shade-hadoop2.6.0.jar 包，到此编译完成

### 四、遇到的问题和建议
1、编译依赖thrift

开始下载源码后，直接无脑编译出错

2、建议编译特定的版本
````
git tag
git checkout apache-carbondata-1.1.1-rc1
````
3、建议配置使用国内maven源
````
<mirror>
<id>nexus</id>
<name>nexus</name>
<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
<mirrorOf>*</mirrorOf>
</mirror>
````
4、期间遇到的一些异常
````
myProject/carbondata/integration/spark/src/main/scala/org/apache/spark/sql/execution/command/carbonTableSchema.scala:179: error: value getOrDefault is not a member of java.util.Map[String,String]
[INFO] possible cause: maybe a semicolon is missing before `value getOrDefault'?
[INFO]       .getOrDefault("sort_scope", CarbonCommonConstants.LOAD_SORT_SCOPE_DEFAULT)
[INFO]        ^
[ERROR]
myProject/carbondata/integration/spark/src/main/scala/org/apache/spark/sql/execution/command/carbonTableSchema.scala:453: error: value getOrDefault is not a member of java.util.Map[String,String]
[INFO]         tableProperties.getOrDefault("sort_scope", sortScopeDefault)
[INFO]                         ^
[ERROR] /myProject/carbondata/integration/spark/src/main/scala/org/apache/spark/sql/execution/command/carbonTableSchema.scala:911: error: value getOrDefault is not a member of java.util.Map[String,String]
[INFO]       .getTableProperties.getOrDefault("sort_scope", CarbonCommonConstants
[INFO]                           ^
[WARNING] two warnings found


[ERROR] Failed to execute goal org.scala-tools:maven-scala-plugin:2.15.2:compile (default) on project carbondata-spark: wrap: org.apache.commons.exec.ExecuteException: Process exited with an error: 1(Exit value: 1) -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.scala-tools:maven-scala-plugin:2.15.2:compile (default) on project carbondata-spark: wrap: org.apache.commons.exec.ExecuteException: Process exited with an error: 1(Exit value: 1) ```
````
