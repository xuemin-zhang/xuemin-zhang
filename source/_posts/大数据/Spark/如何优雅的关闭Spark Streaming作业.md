---
title: 如何优雅的关闭Spark Streaming作业
date: 2017-06-20 16:48:05
tags: [其他]
categories: [其他]
---
Spark Streaming 应用定位是长期执行的。但如何优雅的关闭它，使正在被处理的消息在作业停止前被妥善处理？很多博文建议我们必须通过JVM关闭的钩子，可在此 查看相关代码。但是，这个方法在新的Spark版本（1.4版本之后）中不能正常工作，并且会引起死锁情况。

目前有两种方式去优雅的关闭Spark Streaming作业。第一种方法是设置spark.streaming.stopGracefullyOnShutdown参数值为true（默认是false）。这个参数在解决Spark优雅关闭的issue中引入。开发者不再需要去调用ssc.stop()函数，只需要向Driver发送SIGTERM信号。在实践中，我们需要如下操作：

在Spark UI上找到Driver进程运行在哪个节点。在Yarn Cluster部署模式下，Driver进程和AM运行在同一个Container。
登陆运行Driver的节点，并且执行ps -ef |grep java |grep ApplicationMaster 去找到进程ID。请注意，你搜索的字符串可能会因为作业或者环境等原因不同。
执行kill -SIGTERM <AM-PID> 命令，发送SIGTERM信号给进程。
在Spark Driver接收到SIGTERM信号后，你会在日志中看到类似如下的消息：

````
17/02/02 01:31:35 ERROR yarn.ApplicationMaster: RECEIVED SIGNAL 15: SIGTERM*
17/02/02 01:31:35 INFO streaming.StreamingContext: Invoking stop(stopGracefully=true) from shutdown hook...
17/02/02 01:31:45 INFO streaming.StreamingContext: StreamingContext stopped successfully**
17/02/02 01:31:45 INFO spark.SparkContext: Invoking stop() from shutdown hook...
17/02/02 01:31:45 INFO spark.SparkContext: Successfully stopped SparkContext...
17/02/02 01:31:45 INFO util.ShutdownHookManager: Shutdown hook called*
````

需要注意，默 spark.yarn.maxAppAttempts默认使用Yarn的yarn.resourcemanager.am.max-attempts的值。而yarn.resourcemanager.am.max-attempts值默认为2。因此，在执行kill命令AM第一次停止后，Yarn将会自动启动另一个AM/Driver。你需要第二次kill掉它。你可以在spark-submit设置--conf spark.yarn.maxAppAttempts=1 ，但是你必须考虑清楚，因为如此配置后Driver失败后将没机会重试。

你不能使用yarn application -kill <applicationid>去kill作业。这个命令不会发送SIGTERM信号给container，而是几乎同时发送SIGKILL信号。SIGTERM和SIGKILL之间的时间间隔可以使用yarn.nodemanager.sleep-delay-before-sigkill.ms (默认 250)去配置。当然，你可以增大该值，但是， 在一定程度上，即使我调整到60000（1分钟），它仍然不起作用。作业的containers几乎是立即被kill掉，并且日志中仅包含如下内容：

````
17/02/02 12:12:27 ERROR yarn.ApplicationMaster: RECEIVED SIGNAL 15: SIGTERM*
17/02/02 12:12:27 INFO streaming.StreamingContext: Invoking stop(stopGracefully=true) from shutdown hook*
````

所以，我不建议使用yarn application -kill <applicationid> 命令去发送SIGTERM信号。

第二个解决方案是以某种方式通知Spark Streaming应用它需要优雅的关闭，而不是使用SIGTERM信号。一种方式是在HDFS上放一个标识文件，Spark Streaming应用周期性的去检测它。如果标识文件存在了，就调用scc.stop(true, true) 。第一个true意思是Spark context需要被停止。第二个true意思是需要优雅的关闭，允许正在处理的消息完成。

至关重要的是，不要在micro-batch的代码中调用ssc.stop(true, true)，试想一下，如果你在微批代码中调用ssc.stop(true, true)，它将等待所有正在被处理的消息完成，包括当前正在执行的微批。但是，当前的微批不会结束，直到ssc.stop(true, true)结束返回。这是一种死锁的情况。所以，你必须在另一个线程中执行标识文件检测和调用ssc.stop(true, true)。我在github上放了一个简单的样例，此样例里我在mian线程中在ssc.start()后执行检测和调用ssc.stop() 。你可以在这里找到源码。当然，使用HDFS标识文件仅仅是一种方法，其他可选择的方法有使用一个单独的线程监听一个socket，启动一个RESTful服务等等。

期待在将来的release中，Spark会考虑更优雅的方案。比如，在Spark UI中可以增加一个按钮，去优雅的停止Spark Streaming作业，这样，我们就不需要凭借定制化的编码或者使用PID和SIGTERM信号了。

翻译：http://blog.parseconsulting.com/2017/02/how-to-shutdown-spark-streaming-job.html
