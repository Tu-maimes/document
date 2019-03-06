---
title: Spark on Yarn 的性能调优
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]

## 运行环境Jar包管理及数据本地性原理调优实践

### 运行环境Jar包管理及数据本地性原理

 1. 在YARN上运行Spark需要在Spark-env.sh或环境变量中设置中配置`HADOOP_CONF_DIR`或者`YARN_CONF_DIR`目录指向Hadoop的配置文件。
 2. 在Spark-default。conf中配置Spark.YARN.jars指向hdfs上的Spark需要的jar包。如果不配置该参数，每次启动Spark程序会将Driver端的SPARK_HOME打包上传分发到各个节点。`spark.yarn.jars.hdfs hdfs://sparklibpath/*`
 3. 通过长期的观察运行状态，设置合理的数据副本数能够更好的做到数据的本地性。
 4. PROCESS_LOCAL:进程本地化。代码与数据在同一个进程中，也就是在同一个Executor中；计算数据的Task由Executor执行，数据在Executor的BlockManager中：性能最好。
 5. NODE_LOCAL:节点本地化：代码和数据在同一个节点中，数据作为一个HDFSblock块，就在节点上，而Task在节点上某个Executor中运行：或者是数据和Task在一个节点上的不同Executor中；数据需要在进程间进行传输。也就是说，数据虽然在同一个Worker中，但不是同一个JVM中。这隐含着进程间移动数据的开销。
 6. NO_PREF：数据没有局部性首选位置，它能从任何位置同等访问。对于Task来说，数据从哪里获取都是一样的。
 7. RACK_LOCAL:机架本地化。数据在不同的服务器上，但在相同的机架。数据需要通过网络在节点之间进行传输。
 8. ANY：数据在不同的服务器及机架上面。这种方式性能最差。
	
	Spark 应用程序本身包含代码和数据两个部分，单机版一般情况下很少考虑到数据的本地性的问题，因为数据就在本地。但基本的程序，数据的本地性有PROCESS_LOCAL和NODE_LOCAL之分，但也应尽量让数据处于PROCESS_LOCAL级别。
	
	通常，读取数据要尽量使用数据以PROCESS_LOCAL或者NODE_LOCAL方式读取。其中，PROCESS_LOCAL还和Cache有关，如果RDD经常用，应将该RDD Cache带内存中。注意，由于Cache是Lazy级别的，所以必须通过Action的触发，才能真正地将该RDD Cache到内存中。
	

### 运行环境jar包管理及数据本地性调优实践

 1. 启动Spark程序时，其他节点会自动下载Jar包并进行缓存，下次启动时如果包没有变化，则会直接读取本地缓存的包，缓存清理间隔在YARN-site.xml通过以下参数配置。`yarn.nodemanager.locallzer.cache.cleanip.interval-ms`
 2. 参数配置的优先级：SparkConf > Spark-Submit=Spark-shell>配置文件(conf/Spark-defaults.conf)
 3. Spark作业中的Executor配置单种数据本地性级别可以等待的空闲时间：默认情况下等待时长是3s

``` scala?linenums
 val  SparkConf = new SparkConf()
 SparkConf.set("Spark.locality.wait","3s")
 SparkConf.set("Spark.locality.wait.node,"3s")
 SparkConf.set("Spark.locality.wait.process","3s")
 SparkConf.set("Spark.locality.wait.rack","3s")
```
在测试时采用Client模式运行，查看Task的数据本地化的级别，适当的调节等待时间，以及整个Spark作业所用的时间，找出最优值。

## Spark on YARN 两种不同的调度模型及其调优

### Spark on YARN 的两种不同类型模型优劣分析

按照Spark应用程序中的Driver分布方式的不同，Spark on YARN 有两种模式：YARN-Client模式、YARN-Cluster模式。

不论是在Spark-Shell或者Spark-Submit中，Driver都运行在启动Spark应用的机器上。在这种情形下，YARN Application Master 仅负责从YARN中请求资源，这就是YARN-Client模式。

另一种方式，Driver自动运行在YARN Container(容器)里，客户端可以从集群中断开或者用于其他作业。这叫作YARN-Cluster模式。

YARN-Client模式适合调试Spark程序，能在控制台输出调试信息，YARN-Cluster模式适合企业生产环境。

### Spark on YARN 的两种不同类型调优实践

在YARN上运行会产生一些复杂情况：

 1. 一个常见的问题是YARN Resource Manager 对 Spark的申请资源的限制。
 2. 一个通用的问题是单个YARN Container的可用资源是固定的。单个Container里面资源有限，即使分配多个Container，当Spark应用在一个YARN Container里面超过了可用内存，就会出现OOM问题。在这种情况下，YARN将终结容器并抛出错误，而且问题的底层原因很难追踪。

## YARN队列资源不足引起的Spark应用程序失败的原因及调优方案

### YARN队列资源不足导致失败的原因剖析

ResourceManager会接收你提交的请求吗？YARN一般把自己的资源分成不同的类型，我们提交的时候会专门提交到分配给Spark的那一组资源。例如，我们提交信息资源：Memory 1000G，Cores 800个。此时你要提交的Spark应用程序可能需要900GB的内存和700个Core，一定会没有问题吗？不一定！

另一种情况是当前的作业可以提交运行，已经消耗了900GB的内存和700个Core，然后又提交了一个消耗500GB内存和300个Core的Spark应用程序，这时资源不够，无法提交。

### 调优方案

YARN队列资源不足引起的Spark应用程序失败解决方案。

 1. 在J2EE中间层，通过线程池技术实现顺利地提交，可让线程池的大小设定为1。
 2. 如果提交的Spark应用程序比较耗时，如均超过10min，而其他的Spark程序都在2min内执行完成，这时可以把Spark拥有的资源进行分类(耗时任务和快速任务)。此时可以使用两个线程池，每个线程池都有一个线程。