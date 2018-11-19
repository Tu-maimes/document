---
title: Flume的介绍 
tags: 作者:汪帅
grammar_cjkRuby: true
---
[toc!]

## 一、Flume的介绍

### 1.1什么是Flume

		可以理解flume是日志收集系统，Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接受方（可定制）的能力。
       当前Flume有两个版本Flume 0.9X版本的统称Flume-og，Flume1.X版本的统称Flume-ng。由于Flume-ng经过重大重构，与Flume-og有很大不同，使用时请注意区分，经过架构重构后，Flume NG更像是一个轻量级的小工具，适应各种方式的日志收集，并支持failover和负载均衡。改动的另一原因是将 Flume 纳入 apache 旗下，cloudera Flume 改名为 Apache Flume。
      Apache Flume是一个分布式，可靠且可用的系统，用于高效地收集，汇总和将来自多个不同源的大量日志数据移动到集中式数据存储区。Apache Flume的使用不仅限于日志数据聚合。 由于数据源是可定制的，Flume可用于传输大量的事件数据，包括但不限于网络流量数据，社交媒体生成的数据，电子邮件消息以及几乎所有可能的数据源。Apache Flume是Apache软件基金会的顶级项目。

### 1.2Flume的结构

 - 数据流模型

		Flume事件被定义为具有字节有效载荷和一组可选字符串属性的数据流单元。 Flume代理是一个（JVM）进程，它承载事件从外部源流向下一个目标（跳）的组件。Flume以agent为最小的独立运行单位。一个agent就是一个JVM。单agent由Source、Sink和Channel三大组件构成。
	![1.1 Flume的数据流结构图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/结构图.jpg)
	
		Flume源消耗由外部源（如Web服务器）传递给它的事件。外部源以Flume源识别的格式向Flume发送事件。例如，Avro Flume源可用于从Avro客户端或其他Flume客户端接收从Avro接收器发送事件的流中的Avro事件。使用Thrift Flume Source可以定义类似的流程，以接收来自Thrift Sink或Flume Thrift Rpc客户端的事件或使用Flume thrift协议生成的任何语言编写的Thrift客户端。当Flume源接收事件时，将其存储到一个或多个频道。渠道是一个被动的商店，保持事件，直到它被一个水槽水槽消耗。文件通道就是一个例子 - 它由本地文件系统支持。接收器从通道中删除事件并将其放入HDFS（通过Flume HDFS接收器）之类的外部存储库，或者将其转发到流中下一个Flume代理（下一个跃点）的Flume源。给定代理中的源和宿与运行在通道中的事件异步运行。
		Flume的数据流由事件(Event)贯穿始终。事件是Flume的基本数据单位，它携带日志数据(字节数组形式)并且携带有头信息，这些Event由Agent外部的Source，比如上图中的Web Server生成。当Source捕获事件后会进行特定的格式化，然后Source会把事件推入(单个或多个)Channel中。你可以把Channel看作是一个缓冲区，它将保存事件直到Sink处理完该事件。Sink负责持久化日志或者把事件推向另一个Source。很直白的设计，其中值得注意的是，Flume提供了大量内置的Source、Channel和Sink类型。不同类型的Source,Channel和Sink可以自由组合。组合方式基于用户设置的配置文件，非常灵活。比如：Channel可以把事件暂存在内存里，也可以持久化到本地硬盘上。Sink可以把日志写入HDFS, HBase，甚至是另外一个Source等等。
		

 - 复杂流结构模型
				![1.2 Flume的复杂流数据结构](https://www.github.com/Tu-maimes/document/raw/master/小书匠/复杂的流.jpg)