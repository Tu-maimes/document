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
 3. 只有一个程序运行的时候，可以把Memory和Cores都调整到最大，这样最大化地使用资源来最快速地完成程序的计算，同时也简化了集群的运维和故障解决。

## Spark on YARN模式下Executor经常被杀死的原因及调优方案

### 原因剖析

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1551855194687.png)
这个异常的信息很明显是内存被用完了，提示建议考虑增加Spark.YARN.Executor.memoryOverhead的配置。

### 调优方案

Spark on YARN 模式下Executor经常被杀死的调优方案可考虑：

 1. 移除RDD缓存操作
 2. 增加该Job的Spark.storage.memoryFraction系数值。
 3. 增加该Job的spark.yarn.Executor.memoryoverhead值。

## YARN-Client模式下网卡流量激增的原因及调优方案

### 原因剖析

YARN集群分成两种节点：
- ResourceManager负责资源的调度。
- NodeManager负责资源的分配、应用程序执行。

通过Spark-Submit脚本使用YARN-Client方式提交，这种模式其实会在本地启动Driver程序。
我们将写的Saprk程序打成Jar包，使用Spark-Submit提交，将Jar包中的main类通过JVM命令启动。JVM的进程其实就是Driver进程，Driver进程启动后，执行我们写的main函数。

ApplicationMaster(ExecutorLauncher)负责Executor的申请：Driver负责Job和Stage的划分，已经Task的创建、分配和调度。

### 调优方案

YARN-Client模式通常只会在测试环境中。

 1. 是否需要添加YARN集群增加一些网络带宽。
 2. 使用YARN-Cluster模式后，就不是本地机器运行Driver负责Task调度了。YARN集群中，有某个节点会运行Driver进程，负责Task调度。

## YARN-Cluster模式下JVM栈内存溢出的原因及调优方案

### 原因剖析

有些Spark作业在YARN-Client模式下是可以运行的，但在YARN-Cluster模式下，会报出JVM的PermGen(永久代)的内存溢出(OOM)。

出现以上问题的原因是：YARN-Client模式下，Driver运行在本地机器上，Spark使用JVM的PermGen的配置，是本地的默认配置128M；但在YARN-Cluster模式下，Driver运行在集群的某个节点上，Spark使用的JVM的PermGen是没有经过配置的，默认是82MB，故有时会出现PermGen Out Of Memory error Log。

### 调优方案

YARN-Cluster模式下JVM栈内存溢出问题的调优方案如下。

 1. 在Spark-Submit脚本中设置PermGen。
`-conf Spark.Driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"`

建议使用如下：解决方法就是在Spark的conf目录中的spark-defaults.conf里，增加对Driver的JVM配置

`spark.driver.extraJavaOptions -XX:PermSize=128M -XX:MaxPermSize=256M`
   
 2. 如果使用Spark SQL，SQL中使用大量的or语句，也可能会报出JVM stack overflow，JVM栈内存溢出，此时可以 吧复杂的SQL简化为多个简单的SQL进行处理。

### Driver的JVM参数

-Xmx，-Xms，如果是yarn-client模式，则默认读取spark-env文件中的SPARK_DRIVER_MEMORY值，-Xmx，-Xms值一样大小；如果是yarn-cluster模式，则读取的是spark-default.conf文件中的spark.driver.extraJavaOptions对应的JVM参数值。
PermSize，如果是yarn-client模式，则是默认读取spark-class文件中的JAVA_OPTS="-XX:MaxPermSize=256m $OUR_JAVA_OPTS"值；如果是yarn-cluster模式，读取的是spark-default.conf文件中的spark.driver.extraJavaOptions对应的JVM参数值。
GC方式，如果是yarn-client模式，默认读取的是spark-class文件中的JAVA_OPTS；如果是yarn-cluster模式，则读取的是spark-default.conf文件中的spark.driver.extraJavaOptions对应的参数值。
以上值最后均可被spark-submit工具中的--driver-java-options参数覆盖。

### Executor的JVM参数
-Xmx，-Xms，如果是yarn-client模式，则默认读取spark-env文件中的SPARK_EXECUTOR_MEMORY值，-Xmx，-Xms值一样大小；如果是yarn-cluster模式，则读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。
PermSize，两种模式都是读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。
GC方式，两种模式都是读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。

### Executor数目及所占CPU个数

如果是yarn-client模式，Executor数目由spark-env中的SPARK_EXECUTOR_INSTANCES指定，每个实例的数目由SPARK_EXECUTOR_CORES指定；如果是yarn-cluster模式，Executor的数目由spark-submit工具的--num-executors参数指定，默认是2个实例，而每个Executor使用的CPU数目由--executor-cores指定，默认为1核。



## Spark on yarn 配置项说明与优化整理

配置于spark-default.conf  

1. #spark.yarn.applicationMaster.waitTries  5    

用于applicationMaster等待Spark master的次数以及SparkContext初始化尝试的次数 (一般不用设置)

 

2.spark.yarn.am.waitTime 100s 

 

3.spark.yarn.submit.file.replication 3

应用程序上载到HDFS的复制份数

 

4.spark.preserve.staging.files    false

设置为true，在job结束后，将stage相关的文件保留而不是删除。 （一般无需保留，设置成false)

 

5.spark.yarn.scheduler.heartbeat.interal-ms  5000

Spark application master给YARN ResourceManager 发送心跳的时间间隔（ms）


6.spark.yarn.executor.memoryOverhead  1000

此为vm的开销（根据实际情况调整)

 

7.spark.shuffle.consolidateFiles  true

仅适用于HashShuffleMananger的实现，同样是为了解决生成过多文件的问题，采用的方式是在不同批次运行的Map任务之间重用Shuffle输出文件，也就是说合并的是不同批次的Map任务的输出数据，但是每个Map任务所需要的文件还是取决于Reduce分区的数量，因此，它并不减少同时打开的输出文件的数量，因此对内存使用量的减少并没有帮助。只是HashShuffleManager里的一个折中的解决方案。

 

8.spark.serializer        org.apache.spark.serializer.KryoSerializer

暂时只支持Java serializer和KryoSerializer序列化方式

 

9.spark.kryoserializer.buffer.max 128m

允许的最大大小的序列化值。

 

10.spark.storage.memoryFraction    0.3

用来调整cache所占用的内存大小。默认为0.6。如果频繁发生Full GC，可以考虑降低这个比值，这样RDD Cache可用的内存空间减少（剩下的部分Cache数据就需要通过Disk Store写到磁盘上了），会带来一定的性能损失，但是腾出更多的内存空间用于执行任务，减少Full GC发生的次数，反而可能改善程序运行的整体性能。

 

11.spark.sql.shuffle.partitions 800

一个partition对应着一个task,如果数据量过大，可以调整次参数来减少每个task所需消耗的内存.

 

12.spark.sql.autoBroadcastJoinThreshold -1

当处理join查询时广播到每个worker的表的最大字节数，当设置为-1广播功能将失效。

 

13.spark.speculation   false

如果设置成true，倘若有一个或多个task执行相当缓慢，就会被重启执行。（事实证明，这种做法会造成hdfs中临时文件的丢失，报找不到文件的错)

 

14.spark.shuffle.manager tungsten-sort

tungsten-sort是一种类似于sort的shuffle方式，shuffle data还有其他两种方式 sort、hash. (不过官网说 tungsten-sort 应用于spark 1.5版本以上）

 

15.spark.sql.codegen true

Spark SQL在每次执行次，先把SQL查询编译JAVA字节码。针对执行时间长的SQL查询或频繁执行的SQL查询，此配置能加快查询速度，因为它产生特殊的字节码去执行。但是针对很短的查询，可能会增加开销，因为它必须先编译每一个查询

 

16.spark.shuffle.spill false

如果设置成true，将会把spill的数据存入磁盘

 

17.spark.shuffle.consolidateFiles true

 我们都知道shuffle默认情况下的文件数据为map tasks * reduce tasks,通过设置其为true,可以使spark合并shuffle的中间文件为reduce的tasks数目。

 

18.代码中 如果filter过滤后 会有很多空的任务或小文件产生，这时我们使用coalesce或repartition去减少RDD中partition数量。

分类: Spark