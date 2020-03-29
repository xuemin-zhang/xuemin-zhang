---
title: 让Spark Streaming在YARN上长时间运行
date: 2016-05-16 16:48:05
copyright:
tags: [Spark]
categories: [大数据]
---
对于长时间运行的Spark Streaming作业，一旦提交到YARN群集便需要永久运行，直到有意停止。任何中断都会引起严重的处理延迟，并可能导致数据丢失或重复。YARN和Apache Spark都不是为了执行长时间运行的服务而设计的。但是，它们已经成功地满足了近实时数据处理作业的常驻需求。成功并不一定意味着没有技术挑战。

这篇博客总结了在安全的YARN集群上，运行一个关键任务且长时间的Spark Streaming作业的经验。您将学习如何将Spark Streaming应用程序提交到YARN群集，以避免在值班时候的不眠之夜。

#### Fault tolerance
在YARN集群模式下，Spark驱动程序与Application Master（应用程序分配的第一个YARN容器）在同一容器中运行。此过程负责从YARN 驱动应用程序和请求资源（Spark执行程序）。重要的是，Application Master消除了在应用程序生命周期中运行的任何其他进程的需要。即使一个提交Spark Streaming作业的边缘Hadoop节点失败，应用程序也不会受到影响。

要以集群模式运行Spark Streaming应用程序，请确保为spark-submit命令提供以下参数：
````
spark-submit --master yarn --deploy-mode cluster
````

由于Spark驱动程序和Application Master共享一个JVM，Spark驱动程序中的任何错误都会阻止我们长期运行的工作。幸运的是，可以配置重新运行应用程序的最大尝试次数。设置比默认值2更高的值是合理的（从YARN集群属性yarn.resourcemanager.am.max尝试中导出）。对我来说，4工作相当好，即使失败的原因是永久性的，较高的值也可能导致不必要的重新启动。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4
````
如果应用程序运行数天或数周，而不重新启动或重新部署在高度使用的群集上，则可能在几个小时内耗尽4次尝试。为了避免这种情况，尝试计数器应该在每个小时都重置。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h
````
另一个重要的设置是在应用程序发生故障之前executor失败的最大数量。默认情况下是max（2 * num executors，3），非常适合批处理作业，但不适用于长时间运行的作业。该属性具有相应的有效期间，也应设置。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h
````
对于长时间运行的作业，您也可以考虑在放弃作业之前提高任务失败的最大数量。默认情况下，任务将重试4次，然后作业失败。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8
Performance
````
当Spark Streaming应用程序提交到集群时，必须定义运行作业的YARN队列。我强烈建议使用YARN Capacity Scheduler并将长时间运行的作业提交到单独的队列。没有一个单独的YARN队列，您的长时间运行的工作迟早将被的大量Hive查询抢占。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8  --queue realtime_queue
````
Spark Streaming工作的另一个重要问题是保持处理时间的稳定性和高度可预测性。处理时间应保持在批次持续时间以下以避免延误。我发现Spark的推测执行有很多帮助，特别是在繁忙的群集中。当启用推测性执行时，批处理时间更加稳定。只有当Spark操作是幂等时，才能启用推测模式。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8  --queue realtime_queue --conf spark.speculation=true
````
#### Security
在安全的HDFS群集上，长时间运行的Spark Streaming作业由于Kerberos票据到期而失败。没有其他设置，当Spark Streaming作业提交到集群时，会发布Kerberos票证。当票证到期时Spark Streaming作业不能再从HDFS写入或读取数据。

在理论上（基于文档），应该将Kerberos主体和keytab作为spark-submit命令传递：
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8  --queue realtime_queue --conf spark.speculation=true  --principal user/hostname@domain --keytab /path/to/foo.keytab
````
实际上，由于几个错误（HDFS-9276, SPARK-11182）必须禁用HDFS缓存。如果没有，Spark将无法从HDFS上的文件读取更新的令牌。
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8  --queue realtime_queue --conf spark.speculation=true  --principal user/hostname@domain --keytab /path/to/foo.keytab --conf spark.hadoop.fs.hdfs.impl.disable.cache=true
````
Mark Grover指出，这些错误只影响在HA模式下配置了NameNodes的HDFS集群。谢谢，Mark。

#### Logging
访问Spark应用程序日志的最简单方法是配置Log4j控制台追加程序，等待应用程序终止并使用yarn logs -applicationId [applicationId]命令。不幸的是终止长时间运行的Spark Streaming作业来访问日志是不可行的。

我建议安装和配置Elastic，Logstash和Kibana（ELK套装）。ELK的安装和配置是超出了这篇博客的范围，但请记住记录以下上下文字段：
````
YARN application id
YARN container hostname
Executor id (Spark driver is always 000001, Spark executors start from 000002)
YARN attempt (to check how many times Spark driver has been restarted)
````
Log4j配置使用Logstash特定的appender和布局定义应该传递给spark-submit命令：
````
spark-submit --master yarn --deploy-mode cluster --conf spark.yarn.maxAppAttempts=4  --conf spark.yarn.am.attemptFailuresValidityInterval=1h --conf spark.yarn.max.executor.failures={8 * num_executors} --conf spark.yarn.executor.failuresValidityInterval=1h --conf spark.task.maxFailures=8  --queue realtime_queue --conf spark.speculation=true  --principal user/hostname@domain --keytab /path/to/foo.keytab --conf spark.hadoop.fs.hdfs.impl.disable.cache=true  --conf spark.driver.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties --conf spark.executor.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties --files /path/to/log4j.properties
````
最后，Spark Job的Kibana仪表板可能如下所示：


#### Monitoring
长时间运行的工作全天候运行，所以了解历史指标很重要。Spark UI仅在有限数量的批次中保留统计信息，并且在重新启动后，所有度量标准都消失了。再次，需要外部工具。我建议安装Graphite用于收集指标和Grafana来建立仪表板。

首先，Spark需要配置为将指标报告给Graphite，准备metrics.properties文件：
````
*.sink.graphite.class=org.apache.spark.metrics.sink.GraphiteSink
*.sink.graphite.host=[hostname]
*.sink.graphite.port=[port] *.sink.graphite.prefix=some_meaningful_name

driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource
executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource
````
#### Graceful stop
最后一个难题是如何以优雅的方式停止部署在YARN上的Spark Streaming应用程序。停止（甚至杀死）YARN应用程序的标准方法是使用命令yarn application -kill [applicationId]。这个命令会停止Spark Streaming应用程序，但这可能发生在批处理中。因此，如果该作业是从Kafka读取数据然后在HDFS上保存处理结果，并最终提交Kafka偏移量，当作业在提交偏移之前停止工作时，您应该预见到HDFS会有重复的数据。

解决优雅关机问题的第一个尝试是在关闭程序时回调Spark Streaming Context的停止方法。
````
sys.addShutdownHook {
    streamingContext.stop(stopSparkContext = true, stopGracefully = true)
}
````
令人失望的是，由于Spark应用程序几乎立即被杀死，一个退出回调函数来不及完成已启动的批处理任务。此外，不能保证JVM会调用shutdown hook。

在撰写本博客文章时，唯一确认的YARN Spark Streaming应用程序的确切方法是通知应用程序关于计划关闭，然后以编程方式停止流式传输（但不是关闭挂钩）。命令yarn application -kill 如果通知应用程序在定义的超时后没有停止，则应该仅用作最后手段。

可以使用HDFS上的标记文件（最简单的方法）或使用驱动程序上公开的简单Socket / HTTP端点（复杂方式）通知应用程序。

因为我喜欢KISS原理，下面你可以找到shell脚本伪代码，用于启动/停止Spark Streaming应用程序使用标记文件：
````
start() {
    hdfs dfs -touchz /path/to/marker/my_job_unique_name
    spark-submit ...
}

stop() {
    hdfs dfs -rm /path/to/marker/my_job_unique_name
    force_kill=true application_id=$(yarn application -list | grep -oe "application_[0-9]*_[0-9]*"`) for i in `seq 1 10`; do application_status=$(yarn application -status ${application_id} | grep "State : \(RUNNING\|ACCEPTED\)") if [ -n "$application_status" ]; then
            sleep 60s else force_kill=false
            break fi
    done
    $force_kill && yarn application -kill ${application_id}
}
````
在Spark Streaming应用程序中，后台线程应该监视标记文件，当文件消失时停止上下文调用
````
streamingContext.stop(stopSparkContext = true, stopGracefully = true)
````
#### Summary
可以看到，部署在YARN上的关键任务Spark Streaming应用程序的配置相当复杂。以上提出的技术，由一些非常聪明的开发人员经过漫长而冗长乏味的迭代学习。最终，部署在高可用的YARN集群上的长期运行的Spark Streaming应用非常稳定。

翻译：http://mkuthan.github.io/blog/2016/09/30/spark-streaming-on-yarn/
