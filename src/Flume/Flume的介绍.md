---
title: Flume的介绍 
tags: 作者:汪帅
grammar_cjkRuby: true
---


## 一、Flume的介绍

### 1.1什么是Flume

 可以理解flume是日志收集系统，Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接受（可定制）的能力。
当前Flume有两个版本Flume 0.9X版本的统称Flume-og，Flume1.X版本的统称Flume-ng。由于Flume-ng经过重大重构，与Flume-og有很大不同，使用时请注意区分，经过架构重构后，Flume NG更像是一个轻量级的小工具，适应各种方式的日志收集，并支持failover和负载均衡。改动的另一原因是将 Flume 纳入 apache 旗下，cloudera Flume 改名为 Apache Flume。
Apache Flume是一个分布式，可靠且可用的系统，用于高效地收集，汇总和将来自多个不同源的大量日志数据移动到集中式数据存储区。Apache Flume的使用不仅限于日志数据聚合。 由于数据源是可定制的，Flume可用于传输大量的事件数据，包括但不限于网络流量数据，社交媒体生成的数据，电子邮件消息以及几乎所有可能的数据源。Apache Flume是Apache软件基金会的顶级项目。

### 1.2Flume的结构

 - 数据流模型

Flume事件被定义为具有字节有效载荷和一组可选字符串属性的数据流单元。 Flume代理是一个（JVM）进程，它承载事件从外部源流向下一个目标（跳）的组件。Flume以agent为最小的独立运行单位。一个agent就是一个JVM。单agent由Source、Sink和Channel三大组件构成。

![1.1 Flume的数据流结构图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/结构图.jpg)
	
Flume源消耗由外部源（如Web服务器）传递给它的事件。外部源以Flume源识别的格式向Flume发送事件。例如，Avro Flume源可用于从Avro客户端或其他Flume客户端接收从Avro接收器发送事件的流中的Avro事件。使用Thrift Flume Source可以定义类似的流程，以接收来自Thrift Sink或Flume Thrift Rpc客户端的事件或使用Flume thrift协议生成的任何语言编写的Thrift客户端。当Flume源接收事件时，将其存储到一个或多个频道。渠道是一个被动的商店，保持事件，直到它被一个水槽水槽消耗。文件通道就是一个例子 - 它由本地文件系统支持。接收器从通道中删除事件并将其放入HDFS（通过Flume HDFS接收器）之类的外部存储库，或者将其转发到流中下一个Flume代理（下一个跃点）的Flume源。给定代理中的源和宿与运行在通道中的事件异步运行。

Flume的数据流由事件(Event)贯穿始终。事件是Flume的基本数据单位，它携带日志数据(字节数组形式)并且携带有头信息，这些Event由Agent外部的Source，比如上图中的Web Server生成。当Source捕获事件后会进行特定的格式化，然后Source会把事件推入(单个或多个)Channel中。你可以把Channel看作是一个缓冲区，它将保存事件直到Sink处理完该事件。Sink负责持久化日志或者把事件推向另一个Source。很直白的设计，其中值得注意的是，Flume提供了大量内置的Source、Channel和Sink类型。不同类型的Source,Channel和Sink可以自由组合。组合方式基于用户设置的配置文件，非常灵活。比如：Channel可以把事件暂存在内存里，也可以持久化到本地硬盘上。Sink可以把日志写入HDFS, HBase，甚至是另外一个Source等等。
		

 - 复杂流结构模型


![1.2 Flume复杂流结构模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/多路复杂.jpg)

	
  Flume允许用户在到达最终目的地之前通过多个代理程序建立多跳流。 它还允许扇入(fan-in)和扇出（fan-out）流，上下文路由和备份路由（故障转移），以用于失败的跳数。
    ==注意==：同一个Source可以将数据存储到多个Channel,实际上是Replication。一个sink只能从一个Channel中读取数据Channel和Sink是一一对应的。==一个Channel只能对应一个Sink，一个Sink也只能对应一个Channel。==
	

 - Flume的可靠性


这些事件(events)在每个代理人（channel）的频道上进行。 然后将事件传递到流中的下一个代理或终端存储库（如HDFS）。 这些事件只有在存储在下一个代理通道或终端存储库中后才会从通道中删除。 这是Flume中单跳消息传递语义如何提供流的端到端可靠性。
Flume使用事务方式来保证事件的可靠传送。 sources和sinks分别在事务中封装由信道提供的事务中放置或提供的事件的存储/检索,这确保了该组事件在流程中可靠地传递（Source和Sink被封装进一个事务。事件被存放在Channel中直到该事件被处理，Channel中的事件才会被移除。这是Flume提供的点到点的可靠机制。）。 在多跳流的情况下，来自前一跳的接收器和来自下一跳的源两者的事务都在运行，以确保数据安全地存储在下一跳的信道中。

 - Flume的可恢复性

事件在Channel中进行，管理从故障恢复。 Flume支持由本地文件系统支持的持久文件通道。 还有一个内存通道，它将事件简单地存储在内存队列中，这个速度更快，但是当代理进程死亡时仍然留在内存通道中的任何事件都不能被恢复。

### 1.3Flume名词的解释

 - Event:

Event是Flume数据传输的基本单元。Flume以事件的形式将数据从源头传送到最终的目的。Event由可选的header和载有数据的一个byte array 构成。可以是日志记录、 avro 对象等

 - Client:

Client 是一个将原始log包装成events并且发送他们到一个或多个agent的实体目的是从数据源系统中解耦Flume，==在flume的拓扑结构中不是必须的 #ec1d0e==。Client实例：flume log4j Appender，可以使用Client SDK（org.apache.flume.api)定制特定的Client。生产数据，运行在一个独立的线程。
 - Agent:

 使用JVM 运行Flume。每台机器运行一个agent，但是可以在一个agent中包含多个sources和sinks。一个Agent包含 source ，channel，sink 和其他组件。它利用这些组件将events从一个节点传输到另一个节点或最终目的地，agent是flume流的基础部分。flume为这些组件提供了配置，声明周期管理，监控支持。
 
 - Source:

从Client收集数据，传递给Channel。Source 负责接收event或通过特殊机制产生event，并将events批量的放到一个或多个Channel，包含==event驱动 #ec1d0e==和==轮询 #ec1d0e==两种类型。必须至少和一个channel关联。
 - Channel:

连接 sources 和 sinks ，这个有点像一个队列，Channel有多种方式：有MemoryChannel, JDBC Channel, MemoryRecoverChannel, FileChannel。MemoryChannel可以实现高速的吞吐，但是==无法保证数据的完整 #ec1d0e==。MemoryRecoverChannel在官方文档的建议上已经建义使用FileChannel来替换。FileChannel保证数据的完整性与一致性。在具体配置FileChannel时，建议FileChannel设置的目录和程序日志文件保存的目录设成不同的磁盘，以便提高效率。中转Event的一个临时存储,保存有source组件传递过来的Event,当sink成功的将event发送到下一个channel或最终目的,event从Channel移除，不同的channel提供的持久化水平是不一样的。
 - Sink:

从Channel收集数据，运行在一个独立线程，Sink在设置存储数据时，可以向文件系统中，数据库中，hadoop中储数据，在日志数据较少时，可以将数据存储在文件系中，并且设定一定的时间间隔保存数据。在日志数据较多时，可以将相应的日志数据存储到Hadoop中，便于日后进行相应的数据分析。负责将event传输到下一跳或最终目的，成功后将event从channel移除，必须作用一个确切的channel。
 - Iterator:

作用于Source，按照预设的顺序在必要地方装饰和过滤events。
 - channel selector：

允许Source基于预设的标准，从所有channel中，选择一个或者多个channel。
 - sink processor：

多个sink 可以构成一个sink group，sink processor 可以通过组中所有sink实现负载均衡，也可以在一个sink失败时转移到另一个。

### 1.4Flume组件的说明

 - Flume Source:

|Source类型|说明|
|---|---|
|Avro Source|支持Avro协议（实际上是Avro RPC），内置支持|    
|Thrift Source|支持Thrift协议，内置支持 |   
|Exec Source|基于Unix的command在标准输出上生产数据 |   
|JMS Source|从JMS系统（消息、主题）中读取数据，ActiveMQ已经测试过  |  
|Spooling Directory Source|监控指定目录内数据变更  |  
|Twitter 1% firehose Source|通过API持续下载Twitter数据，试验性质   | 
|Netcat Source|监控某个端口，将流经端口的每一个文本行数据作为Event输入 |   
|Sequence Generator Source|序列生成器数据源，生产序列数据   | 
|Syslog Sources|读取syslog数据，产生Event，支持UDP和TCP两种协议    |
|HTTP Source|基于HTTP POST或GET方式的数据源，支持JSON、BLOB表示形式   | 
|Legacy Sources|兼容老的Flume OG中Source（0.9.x版本）|

 - Flume  Channel：

|Channel类型|说明
|---|---|
|MemoryChannel|Event数据存储在内存中|
|JDBCChannel|Event数据存储在持久化存储中，当前FlumeChannel内置支持Derby|
|FileChannel|Event数据存储在磁盘文件中|
|SpillableMemoryChannel|Event数据存储在内存中和磁盘上，当内存队列满了，会持久化到磁盘文件（当前试验性的，不建议生产环境使用）|
|PseudoTransactionChannel|测试用途|
|CustomChannel|自定义Channel实现|

 - Flume Sink

|Sink类型|说明
|---|---|
|HDFS Sink|数据写入HDFS|
|Logger Sink |数据写入日志文件|
|Avro Sink|数据被转换成Avro Event，然后发送到配置的RPC端口上|
|Thrift Sink |数据被转换成Thrift Event，然后发送到配置的RPC端口上|
|IRC Sink |数据在IRC上进行回放|
|File Roll Sink |存储数据到本地文件系统|
|Null Sink|丢弃到所有数据|
|HBase Sink  |数据写入HBase数据库|
|Morphline Solr Sink  |数据发送到Solr搜索服务器（集群）|
|ElasticSearch Sink|数据发送到Elastic Search搜索服务器（集群）|
|Kite Dataset Sink |写数据到Kite Dataset，试验性质的|
|Custom Sink |自定义Sink实现|



## 二、Flume的安装

 1. 安装Flume
``` javascript
# useradd flume

# wget mirrors.hust.edu.cn/apache/flume/1.8.0/apache-flume-1.8.0-bin.tar.gz
# tar zxf apache-flume-1.8.0-bin.tar.gz -C /home/flume/
# ln -s /home/flume/apache-flume-1.8.0-bin /home/flume/flume
```

 2. 配置环境变量
 

``` javascript
# vim /etc/profile

#添加Flume java path到profile

####flume java path####
export JAVA_HOME=/usr/java/jdk
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
export FLUME_HOME=/home/flume/flume
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$FLUME_HOME/bin:$PATH

# source /etc/profile
# java -version

openjdk version "1.8.0_191"
OpenJDK Runtime Environment (build 1.8.0_191-b12)
OpenJDK 64-Bit Server VM (build 25.191-b12, mixed mode)


#查看Flume的版本

# flume-ng version


Flume 1.5.2.2.6.5.0-292
Source code repository: https://git-wip-us.apache.org/repos/asf/flume.git
Revision: 2f89159520e7c477abd961d4e8b3b9e50597f1c9
Compiled by jenkins on Fri May 11 07:52:15 UTC 2018
From source with checksum b50e3a2d9ed95bb61cd587feaa12b813


```

 3. 修改flume-env.sh的JAVA_HOME

``` javascript

# export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export JAVA_HOME=/usr/java/jdk
```

 4. Flume-ng的命令行参数

|commands|说明|
|---|---|
|    help    |帮助显示这个帮助文本|
|    agent   |代理运行Flume代理|
|    avro-client  |运行一个avro Flume客户端|
|    version    |版本显示Flume版本信息|
|全局 options：|说明|
|    --conf,-c <conf>      |使用<conf>目录中的配置|
|    --classpath,-C <cp>   |附加类路径|
|    --dryrun,-d           |实际上不启动Flume，只是打印命令|
|    --plugins-path <dirs>  |以冒号分隔的plugins.d目录列表。请参阅用户指南中的plugins.d部分以获取更多详细信息。默认:$FLUME_HOME/plugins.d|
|    -Dproperty=value      |设置Java系统属性值|
|    -Xproperty=value      |设置一个Java -X选项|
|agent options：|说明|
|    --name,-n <name>     |这个agent的名字（必填）|
|    --conf-file,-f <file>   |指定一个配置文件（如果缺少-z则需要）|
|    --zkConnString,-z <str>  |指定要使用的ZooKeeper连接（如果缺少-f，则需要）|
|    --zkBasePath,-p <path>   |在ZooKeeper中为代理配置指定基本路径|
|    --no-reload-conf    |如果更改，不要重新加载配置文件|
|    --help,-h    |显示帮助文本|
|avro-client options:|说明|
|    --rpcProps,-P <file>   |带有服务器连接参数的RPC客户端属性文件|
|    --host,-H <host>       |将要发送events(事件）的主机名|
|    --port,-p <port>       |avro源的端口|
|    --dirname <dir>        |流到avro源流到的目标目录|
|    --filename,-F <file>   |文本文件流到avro源（默认：标准输入）|
|    --headerFile,-R <file>  |包含事件标题作为每个新行上的键/值对的文件|
|    --help,-h    |显示帮助文本|

**==注意 #ec1d0e==**:

 - 如果指定了<conf>目录，则==始终将其包含在类路径==中。
 - --rpcProps或者--host和--port都必须指定。



 5.测试官方例子
``` javascript
# example.conf: A single-node Flume configuration

# Name the components on this agent
a1.sources = r1     
#定义sources源的名称
a1.sinks = k1       
#定义存储介质的名称
a1.channels = c1    
#定义channels通道的名称

# Describe/configure the source
a1.sources.r1.type = netcat    
#source类型，监控某个端口，将流经端口的每一个文本行数据作为Event输入
a1.sources.r1.bind = localhost   
#监听IP地址
a1.sources.r1.port = 6666       
#监听端口

# Describe the sink
a1.sinks.k1.type = logger       
#Sink类型是logger，也就是数据是日志类型

# Use a channel which buffers events in memory
a1.channels.c1.type = memory    
#channel通道c1类型是内存类型，也就是event数据存储在内存中，然后发送events
a1.channels.c1.capacity = 1000   
#存储在通道c1中的事件的最大数量
a1.channels.c1.transactionCapacity = 100   
#channel将从一个channel获得的最大事件数量或每次传输给予一个sink的事件数量

# Bind the source and sink to the channel
a1.sources.r1.channels = c1    
#选择从channels通道c1来发送events
a1.sinks.k1.channel = c1      
#选择从channels通道c1来接收events
```

 6. 启动Flume进程：

``` javascript
flume-ng agent -c conf -f /etc/flume/conf/flume-conf.properties.template -n agent

#配置此参数不生效可能与log4j2.xml有关,官网默认是log4j.properties
#-Dflume.root.logger=INFO,console

```

 7.打开终端进行测试

``` javascript
# telnet localhost 6666 
# tail -f /var/log/flume/flume.log
```

 8. Flume不写日志：

![2.1 Flume的log4j未生效](https://www.github.com/Tu-maimes/document/raw/master/小书匠/log4j未生效.jpg)
解决方案：
 - . 在flume的安装下启动，包含Flume配置文件的conf目录。
 - . 在执行Flume脚本时的命令行参数时 -c 指定Flume的配置文件的目录。

 9. 设置Flume的agent的环境变量：

