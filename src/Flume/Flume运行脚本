#****************************************************************************************
#*** 说明文档: 自定义Flume组件总体架构阐述!
#*** 编 写 人:  wangshuai
#*** 编写日期:  2018-09-21
#*** 修 改 人:
#*** 修改日期:
#*** 备    注:
#*****************************************************************************************


1.Flume监控Linux上的路径

		目录	/data/gz_interface/ 	
		
		机器 192.168.102.114
		
2.Flume上传文件到hdfs的路径为

		/yss/guzhi/interface/

3.Flume运行脚本
		
# 指定Agent的组件名称
a1.sources = r1
a1.sinks = k1
a1.channels = c1
 
# 指定Flume source(要监听的路径)
a1.sources.r1.type = com.yss.taildirsource.TaildirSource
a1.sources.r1.positionFile = /data/test/ws/taildir_position.json
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /data/gz_interface/^((?!\.xls[x|d]$).)*$
a1.sources.r1.filegroups.f1.headerKey1 = value1
a1.sources.r1.recursiveDirectorySearch = true
 
# 指定Flume sink
a1.sinks.k1.type = com.yss.hdfssink.HDFSEventSink
a1.sinks.k1.hdfs.path = /yss/guzhi/interface/
a1.sinks.k1.hdfs.filePrefix = %{fileName}
a1.sinks.k1.hdfs.fileSuffix = .csv
a1.sinks.k1.hdfs.rollCount = 0
a1.sinks.k1.hdfs.fileType  = DataStream
a1.sinks.k1.hdfs.writeFormat  = Text
a1.sinks.k1.hdfs.rollSize = 0
a1.sinks.k1.hdfs.idleTimeout  = 5
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.rollInterval = 0
a1.sinks.k1.hdfs.useLocalTimeStamp = true
 
# 指定Flume channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000
 
# 绑定source和sink到channel上
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1


4.Flume后台运行指令

	nohup	flume-ng agent -c conf -f testsource_1.conf --name a1
	
	
5.开发中对集群Flume下添加的jar文件

		<dependency>
            <groupId>com.linuxense</groupId>
            <artifactId>javadbf</artifactId>
            <version>0.4.0</version>
		</dependency>
		
        <dependency>
            <groupId>org.json</groupId>
            <artifactId>json</artifactId>
            <version>20180813</version>
        </dependency>

6.自定义的FlumeSource
       com.yss.flume.taildir.TaildirSource
      
7.自定义的FlumeSink
       com.yss.flume.hdfssink.HDFSEventSink
      
8.需求
    1、自定义source和sink
    2、在source端解析dbf和xml以及tsv后缀的文件转换成csv中间分割符为逗号
    3、sink保留原文件名字后缀名为.csv
    4、原文件传输完成之后，文件名加一个时间戳以及.xlsd后缀
    5、传到hdfs上的文件中间状态加个.TMP后缀
    6、支持监控多级目录的递归,各级目录都可以监控

	
