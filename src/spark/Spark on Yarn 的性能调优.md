---
title: Spark on Yarn 的性能调优
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]


## 运行环境Jar包管理及数据本地性原理

 1. 在YARN上运行Spark需要在Spark-env.sh或环境变量中设置中配置HADOOP_CONF_DIR或者YARN_CONF_DIR目录指向Hadoop的配置文件。
 2. 在Spark-default。conf中配置Spark.YARN.jars指向hdfs上的Spark需要的jar包。如果不配置该参数，每次启动Spark程序会将Driver端的SPARK_HOME打包上传分发到各个节点。