---
title: Spark on Yarn 的性能调优
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]


## 运行环境Jar包管理及数据本地性原理

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
	

## 运行环境jar包管理及数据本地性调优实践

 1. 启动Spark程序时，其他节点会自动下载Jar包并进行缓存，下次启动时如果包没有变化，则会直接读取本地缓存的包，缓存清理间隔在YARN-site.xml通过以下参数配置。`yarn.nodemanager.locallzer.cache.cleanip.interval-ms`
 2. 参数配置的优先级：