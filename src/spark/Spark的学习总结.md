---
title: Spark的学习总结
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]


# RPC调用的性能模型分析

## 传统RPC调用性能的三宗罪

 1. 兼容性
 2. 可靠性
 3. 序列化后的码流太大

## 高性能的三个主题

 1. 传输
 2. 协议
 3. 线程

![RPC调用性能三要素](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543844794461.png)

# Netty高性能之道
## 异步非阻塞通信

 1. IO多路复用技术。
 2. 低负载、低并发的应用使用同步阻塞IO以降低编程复杂度。
 3. 对于高并发、高负载的应用使用NIO的非阻塞模式进行开发。

![NIO的多路复用模型图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543845950532.png)

 1. Netty架构按照Reactor模式设计和实现的架构图

![NIO服务端通信系列图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543846800281.png)

 2. 客服端通信序列图

![客服端通信序列图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543847054405.png)

Netty的IO线程NioEventLoop由于聚合了多路复用器Selector,可以同时并发处理上千个客服端Channel,由于读写都是非阻塞的,这就可以提升IO线程运行效率，避免频繁阻塞导致的线程挂起。由于Netty采用异步通信模式，一个IO可以并发处理多个客服端的请求操作。传统同步IO——连接——线程模型。架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

## 零拷贝

Netty的零拷贝主要体现在三个方面

 1. 直接使用堆外内存,减少步骤
 2. 提供Buffer组合对象,避免合并开销
 3. 直接写Channel,避免循环write导致内存的拷贝问题

## 内存池

 - JVM虚拟机和JIT即时编译技术的发展，对象的分配和回收时候是个非常轻量级的工作。
 - 对于堆外直接内存的分配和回收，是一件耗时的操作。为了尽量重用缓冲区，Netty提供基于内存池的缓冲重用机制。

Netty的多种内存管理策略，在启动辅助类中配置相关参数，可以实现个性化定制。

 1. 使用内存池分配器创建直接内存缓冲区
 2. 使用非堆内存分配器创建的直接内存缓冲区

> 通过测试发现：采用内存池的ByteBuf比朝生夕灭的ByteBuf性能更优异。

## 高效的Reactor线程模型

常用的Reactor线程模型有三种：

 1. Reactor单线程模型
 2. Reactor多线程模型
 3. 主从Reactor多线程模型

### 单线程模型

Reactor单线程模型,指的是所有的IO操作都在一个NIO线程上面完成,NIO线程的职责如下:

- 作为NIO服务端,接受客服端的TCP连接
- 作为NIO客服端,向服务端发起TCP连接
- 读取通信对端发送消息或者应答消息

![Reactor单线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543851236858.png)


Reactor模式使用的是异步非阻塞IO,所有的IO操作都不会导致阻塞,理论上一个线程可以独立处理所有IO相关的操作。


#### 单线程的不足有一下三点

 1. NIO线程高负载性能不足，无法处理海量的消息编码、解码、读取和发送。
 2. 高负载导致的消息积压处理超时，达到NIO线程的性能瓶颈。
 3. 可靠性太低，单点故障。

### 多线程模型

多线程与单线程最大的区别是有一组NIO线程处理IO操作。

![Reactor 多线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543929465555.png)

Reactor多线程模型的特点：

 1. 有专门的NIO线程Acceptor线程用于监听服务端，接收客户端的TCP连接请求。
 2. 网络IO操作读、写等由一个NIO线程池负责，线程池也可以采用标准的JDK线程池实现，它包含一个任务队列和N个可用的线程，由这些NIO线程负责消息的读取、解码和发送。
 3. 1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，防止发生并发操作问题。

### Reactor主从多线程模型

利用主从NIO线程模型，可以解决1个服务端监听高负载的问题。

![Reactor主从多线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543931538038.png)

## 无锁化的串行设计理念

为了避免锁竞争导致的性能下降,通过实现串行化设计,即消息的处理尽可能的在一个线程里完成,避免线程的切换,导致的线程的竞争和同步锁。


## 高效的并发编程

Netty的高效并发编程主要体现在如下几点：

 1. volatile的大量、正确的使用。
 2. CAS和原子类的广泛使用。
 3. 线程安全容器的使用。
 4. 通过读写锁提升并发性能。

## 高性能的序列化框架

 1. 序列化后的码流大小（网络带宽的占用）
 2. 序列化&反序列化的性能（CPU资源占用）
 3. 是否支持跨语言（异构系统的对接和开发语言切换）

Netty支持用户通过扩展Netty的编码接口实现其它高性能的框架。

## 灵活的TCP参数配置能力

合理的参数配置对性能的影响

 1. SO_RCVBUF和SO_SNDBUF:通常建议值为128k或者256k;
 2. SO_TCPNODELAY:NAGLE算法组合缓冲区小的数据包,提高网络的利用率,对时延敏感的要关闭该优化.
 3. 软中断

# Netty可靠性分析

## 网络通信类故障

### 客服端连接超时

Netty会在发起连接的时候,根据超时的时间来创建ScheduledFuture挂载在Reactor线程上，用于定时监测是否发生连接超时，连接超时关闭连接，反之取消定时任务。

### 通信端强制关闭

在NIO的编程中，会出现由于句柄没有被及时关闭导致的功能和可靠性问题。

 1. ＩＯ的读写操作并非集中在Ｒｅａｃｔｏｒ线程内部，用户的定制行为导致的ＩＯ操作的外逸。
 2. 异常分支没有考虑到，由外部环境的诱因导致程序进入这些分支，引起故障。
 

## 链路的有效检测

心跳检测机制分三个层面：

 1. TCP 层面的心跳检测，即 TCP 的 Keep-Alive 机制，它的作用域是整个 TCP 协议栈；  
 2. 协议层的心跳检测，主要存在于长连接协议中。
 3. 应用层的心跳检测，它主要由各业务产品通过约定方式定时给对方发送心跳消息实
现。 

![心跳检测机制](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544090172357.png)

不同的协议，心跳检测机制也存在差异，归纳起来主要分为两类： 
1)  Ping-Pong 型心跳：由通信一方定时发送 Ping 消息，对方接收到 Ping 消息之后，立
即返回 Pong 应答消息给对方，属于请求-响应型心跳； 
2)  Ping-Ping 型心跳：不区分心跳请求和应答，由通信双方按照约定定时向对方发送心
跳 Ping 消息，它属于双向心跳。 
心跳检测策略如下： 
1)  连续 N 次心跳检测都没有收到对方的 Pong 应答消息或者 Ping 请求消息，则认为链
路已经发生逻辑失效，这被称作心跳超时； 
2)  读取和发送心跳消息的时候如何直接发生了 IO 异常，说明链路已经失效，这被称为
心跳失败。 
无论发生心跳超时还是心跳失败，都需要关闭链路，由客户端发起重连操作，保证
链路能够恢复正常。


Netty 提供的空闲检测机制分为三种： 
1)  读空闲，链路持续时间 t 没有读取到任何消息； 
2)  写空闲，链路持续时间 t 没有发送任何消息； 
3)  读写空闲，链路持续时间 t 没有接收或者发送任何消息。 

## Reactor线程的保护

Reactor 线程是 IO 操作的核心，NIO 框架的发动机，一旦出现故障，将会导致挂载
在其上面的多路用复用器和多个链路无法正常工作。因此它的可靠性要求非常高。 

## 异常处理

尽管 Reactor 线程主要处理IO操作，发生的异常通常是 IO 异常，但是，实际上在一
些特殊场景下会发生非 IO 异常，如果仅仅捕获 IO 异常可能就会导致Reactor线程跑飞。
为了防止发生这种意外，在循环体内一定要捕获 Throwable，而不是 IO 异常或者
Exception。

处理的核心理念就是： 

1)  某个消息的异常不应该导致整条链路不可用； 
2)  某条链路不可用不应该导致其它链路不可用； 
3)  某个进程不可用不应该导致其它集群节点不可用。 

## 死循环保护

死循环是可检测、可预防但是无法完全避免的。Reactor 线程通常处理
的都是 IO相关的操作，因此我们重点关注IO 层面的死循环。 
JDK NIO 类库最著名的就是  epoll bug 了，它会导致 Selector 空轮询，IO 线程 CPU 
100%，严重影响系统的安全性和可靠性。 

Netty 的解决策略： 
1)  根据该 BUG 的特征，首先侦测该BUG 是否发生； 
2)  将问题 Selector 上注册的Channel 转移到新建的 Selector上； 
3)  老的问题 Selector 关闭，使用新建的Selector 替换。 

## 优雅退出

Java 的优雅停机通常通过注册JDK的ShutdownHook 来实现，当系统接收到退出指
令后，首先标记系统处于退出状态，不再接收新的消息，然后将积压的消息处理完，最后
调用资源回收接口将资源销毁，最后各线程退出执行。 

## 内存保护

### 缓冲区的内存泄漏保护

为了提升内存的利用率，Netty 提供了内存池和对象池。但是，基于缓存池实现以后
需要对内存的申请和释放进行严格的管理，否则很容易导致内存泄漏。    

### 缓冲区内存溢出保护 

缓冲区的创建方式通常有两种： 
1)  容量预分配，在实际读写过程中如果不够再扩展； 
2)  根据协议消息长度创建缓冲区;设置缓冲区的最大长度。

# SparkCore

## Spark的第二代计算引擎Tungsten

 钨丝计划主要从三个方面解决性能问题:
 

 1. 统一内存管理模型和二进制处理。
 2. 基于缓存感知计算。
 3. 代码生成。

Spark将内存分为四个部分：

 1. 堆上的存储内存
 2. 堆外的存储内存
 3. 堆上的执行内存
 4. 堆外的执行内存

## Spark核心功能

### 基础设施

在Spark中有很多基础设施，被Spark中的各种组件广泛使用。包括Spark配置（SparkConf）、Spark内置的RPC框架、事件总线、度量系统。

 1. SparkConf用于管理Spark应用程序的各种配置信息.
 2. Spark内置的RPC框架使用Netty实现,有同步和异步的多种实现,Spark各个组件间的通信依赖于此RPC框架.
 3. 如果说RPC框架是跨机器节点不同组件间的通信设施,那么事件总线就是SparkContext内部各个组件间使用事件-------监听器模式异步调用的实现.
 4. 度量系统由Spark中的多种度量源(Source)和多种度量输出(Sink)构成,完成对Spark集群中各个组件运行期状态的监控.

### SparkContext
Spark应用程序的提交与执行都离不开SparkContext的支持.在正式提交应用程序之前,首先需要初始化SparkContext.SparkContxt隐藏了网络通信、分布式部署、消息通信、存储体系、计算引擎、度量系统、文件服务、WebU等内容。

### SparkEnv

Spark执行环境SparkEnv是Spark中的Task运行所必须的组件。SparkEnv内部封装了RPC环境（RPCEnv）、序列化管理器、广播管理器（BroadcastManager）、map任务输出跟踪器（MapOutputTracker）、存储体系、度量系统（MetricsSystem）、输出提交协调器（OutputCommitCoordinator）等Task运行所需的各种组件。

### 存储体系
Spark优先考虑使用各节点的内存作为存储，当内存不足时会考虑使用磁盘，这极大地减少了磁盘IO，提升任务的执行效率。Spark的内存存储空间与执行存储空间被定义为软边界，二者可以相互调用资源，提高了资源的利用效率，又提高了Task的执行效率。Spark可以直接操作操作系统的内存，直接操作系统内存，空间的分配和释放也更迅速。

### 调度系统

调度系统主要由DAGScheduler和TaskScheduler组成，它们都内置在SparkContext中。DAGScheduler负责创建Job、将DAG中的RDD划分到不同的Stage、给Stage创建对应的Task、批量提交Task等功能。TaskSchdule负责按照FIFO或者FAIR等调度算法对Task进行调度;为Task分配资源;将Task发送到集群管理器的当前应用的Executor上,由Executor负责执行等工作。Spark增加了SparkSession和DataFrame的API，SparkSession底层实际依然依赖于SparkContext。

### 计算引擎

计算引擎由内存管理器（MemoryManager）、Tungsten、任务内存管理器（TaskMemory-Manager）、Task、外部排序器（ExternalSorter）、Shuffle（ShuffleManager）等组成。MemoryManager除了对存储体系中的存储内存提供支持和管理外，还为计算引擎中的执行内存提供支持和管理。Tungsten除用于存储外，也可以用于计算或者执行。TaskMemoryManager对分配给单个Task的内存资源进行更细粒度的管理和控制.ExternalSorter用于在map端或reduce端对shuffleMapTask计算得到的中间结果进行排序、聚合等操作。ShuffleManager用于将各个分区对应的ShuffleMapTask产生的中间结果持久化到磁盘,并在reduce端按照分区远程拉取ShuffleMapTask产生的中间结。

## Spark扩展功能

### SparkSQL

Spark Sql的过程可以总结为:首先使用Sql语句解析器(SqlParser)将Sql转换为语法树(Tree),并且使用规则执行器(RuleExecutor)将一系列规则(Rule)应用到语法树,最终生成物理执行计划并执行。其中包括语法分析器（Analyzer）和优化器（Optimizer）。Hive的执行过程与SQL类似。

### Spark Streaming

SparkStreaming与Apache Stom类似。Dstream是Spark Streaming中国所有数据流的抽象，Dstream可以被组织为DStream Graph。Dstream本质上由一系列连续的RDD组成。

### GraphX

Spark提供的分布式图计算框架。GraphX主要遵循整体同步并行计算模式。GraphX目前一封装了最短路径、网页排名、连接组件、三角关系等算法实现。

### MLlib

Spark提供机器学习框架。

## Spark模型设计

### Spark编程模型

Spark应用程序从编程到提交、执行、输出的整个过程。

![代码执行过程](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544275483636.png)


![程序执行的流程](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544406085854.png)


### RDD详解

#### RDD的定义及五大特性的剖析

RDD 是分布式内存的一个抽象概念，是一种高度受限的共享内存模型，即 RDD 是只读的记录分区的集合，能横跨集群所有节点并行计算，是一种基于工作集的应用抽象。
RDD 底层存储原理：其数据分布存储于多台机器上，事实上，每个 RDD 的数据都以 Block的形式存储于多台机器上，每个 Executor 会启动一个 BlockManagerSlave ， 并管理一部分Block；而Block 的元数据由 Driver 节点上的 BlockManagerMaster 保存， BlockManagerSlave生成 Block 后向 BlockManagerMaster 注册该 Block, BlockManagerMaster 管理 RDD 与 Block的关系，当即RDD 不再需要存储的时候，将向 BlockManagerSlave 发送指令删除相应的 Block。BlockManager 管理 RDD 的物理分区，每个 Block 就是节点上对应的一个数据块，可以存储在内存或者磁盘上。而 RDD 中的 Pattition 是一个逻辑数据块，对应相应的物理块 Block。
本质上， 一个朋RDD在代码中相当于数据的一个元数据结构，存储着数据分区及其逻辑结构
映射关系，存储着 RDD 之前的依赖转换关系。

> 数据本地性判断:BlockManagerMaster 会持有整个 Application 的 Block 的位置、Block 所占用的存储空间等元数据信息，在 Spark 的 Driver 的DAGScheduler 中，就是通过这些信息来确认数据运行的本地性的。


RDD作为泛型的抽象的数据结构，支持两种计算操作算子：Transformation（变换）与Action（行动）。且RDD的写操作是粗粒度的，读操作既可以是粗粒度的，也可以是细粒度的。

##### 分区列表

没一个分区都会被一个计算任务(Task)处理,分区数决定并行计算数量。

##### 每一个分区都有一个计算函数

Spark的Rdd计算函数是以分片为基本单位的，每个Rdd都会实现compute函数。

##### 依赖其他RDD

RDD之间的依赖有两种：窄依赖(NarrowDependency）、宽依赖（WideDependency）。RDD是Spark的核心数据结构，通过RDD的依赖关系形成调度关系。通过对RDD的操作形成整个Spark程序。

##### key-value数据类型的RDD 分区器
key-value数据类型的RDD分区器控制分区策略和分区数。每个key-value形式的RDD都有Partitioner属性，它决定了RDD如何分区。当然，Partition的个数还决定每个Stage的Task个数。RDD的分片函数，想控制RDD的分片函数的时候可以分区（Partitioner）传入相关的参数。

##### 每个分区都有一个优先位置列表
它会存储每个Partition的优先位置，对于一个HDFS文件来说，就是每个Partition块的位置。观察运行spark集群的控制台会发现Spark的具体计算，具体分片前，它已经清楚地知道任务发生在什么节点上，也就是说，任务本身是计算层面的、代码层面的，代码发生运算之前已经知道它要运算的数据在什么地方，有具体节点的信息。这就符合大数据中数据不动代码动的特点。

##### RDD源码的剖析

``` scala
/** 
*  : :  DeveloperApi 
＊通过子类实现给定分区的计算
*/ 
@DeveloperApi 
def  compute(split:  Partition， context : TaskContext ）：Iterator[T]
/** 
*  通过子类实现，返回一个RDD分区列表，这个方法仅只被调用一次，它是安全地执行一次
*  耗时计算
*  该数组中的分区必须满足以下属性
* rdd .partitions.zipWithindex.forall {case(partition,  index)  => 
*  partition.  index  ==  index  }
*/ 
protected  def  getPartitions:  Array [Partition] 
/**
*返回对父 ROD 的依赖列表，这个方法仅只被调用一次，它是安全地执行一次耗时计算
*/
protected  def  getDependencies:  Seq [Dependency[_]]  =  deps 
/**
*可选的，指定优先位置，输入参数是 split 分片，输出结果是一组优先的节点位置
*/
 protected def getPreferredLocations(split: Partition): Seq[String] = Nil
 /*可选地由子类覆盖，以指定它们的分区方式*/
 @transient val partitioner: Option[Partitioner] = None
```
TaskContext 是读取或改变执行任务的环境，用 org.apache.spark.TaskContext.get() 
可返回当前可用的 TaskContext ,可以调用内部的函数访问正在运行任务的环境信息。

##### Partition源码的剖析

Partitioner 是一个对象，定义了如何在 key-Value 类型的 RDD 元素中用 Key 分区，从 0 到
nurnPartitions-1 区间内映射每一个 Key 到 Partition ID 。 Partition 是一个 RDD 的分区标识符。

``` scala
/**
 * RDD中分区的标识符。
 */
trait Partition extends Serializable {
  /**
   * 获取分区在其父RDD中的索引
   */
  def index: Int

  // 最好的默认实现HashCode
  override def hashCode(): Int = index

  override def equals(other: Any): Boolean = super.equals(other)
}
```

#### DataSet的定义及内部机制

DataSet 是可以并行使用函数或关系操作转换特定域对象的强类型集合。DataSet中可用的算子分为转换算子和行动算子.DataSet表示一个逻辑计划,描述了生成数据所需的计算.当行动算子被触发时,Spark查询优化器将优化逻辑计划,生成一个并行、分布式有效执行的物理计划。 内部使用编码器将特定类型T转换为Spark的内部类型。二进制数据占用更少的内存以及优化的数据处理效率。可以使用schema函数来查看了解数据的内部二进制结构.
两种创建DataSet数据集的方法:

 1. SaprkSession中使用的read方式
 2. DataSet通过现有的DataSet进行转换


#### RDD 弹性特性七个方面解析

RDD作为弹性分布式数据集,弹性具体体现在七个方面:

 1. 自动进行内存和磁盘数据的切换
 2. 基于Lineage(血统)的高效容错机制
 3. Task如果失败会自动进行特定次数的重试
 4. Stage如果失败会自动进行特定次数的重试
 5. CheckPoint和persist(检查点和持久化),可主动或被动触发(RDD 根据 useDisk 、 useMemory、 useOffHeap 、deserialized 、 replication 5 个参数的组合提供了常用的 12 种基本存储)
 6. 数据调度弹性,DAGSchedule、TASKSchedule和资源管理无关
 7. 数据分片的高度弹性(coalesce)

### RDD依赖关系

##### 依赖划分原则

RDD包含一个或者多个分区，每个分区实际是一个数据集合的片段。在构建DAG的过程中，会将RDD用依赖关系串联起来。每个RDD都有其依赖，这些依赖分为窄依赖与宽依赖两种。从功能角度讲，窄依赖会划分懂啊同一个Stage中，这样就能以管道的方式迭代执行，宽依赖依赖的分区Task不止一个，所以往往需要跨节点传输数据。从容灾的角度讲，恢复计算结果的方式不同。窄依赖只需要重新执行父RDD的丢失分区的计算即可恢复，而宽依赖则需要考虑恢复所有父RDD的丢失分区。
Rdd的血缘关系、Stage划分的角度来看，Rdd构成的DAG经过DAGScheduler调度后的模型图。


![DAGScheduler由Rdd构成的DAG进行调度](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544406837397.png)

解释了依赖划分的原因，实际也解释了为什么要划分Stage这个问题。





### Spark基本架构


从部署的角度来看，Spark集群由集群管理器（Cluster Manager）、工作节点（Worker）、执行器（Executor）、驱动器（Driver）、应用程序（Application）等部分组成。

![Spark 基本架构图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544408271684.png)



#### Cluster Manager

Spark的集群管理器，主要负责对整个集群的资源分配与管理。Cluster Manager在YARN部署模式下为ResourceManager。ClusterManager分配的资源属于一级分配，将各个worker上的内存、CPU等资源分配给Application，并不负责对executor的资源分配。在Standalone部署模式下的Master会直接给Application分配内存、CPU及Executor等资源。

#### Worker

Spark的工作节点。在YARN部署模式下实际由NodeManager替代。Worker节点主要负责以下工作：
- 将自己的内存、CPU等资源通过注册机制告知Cluster Manager;
- 创建Executor;
- 分配资源和任务给Executor;
- 同步资源信息、Executor状态信息给Cluster manager等.

#### Executor

执行计算任务的一线组件.主要负责任务的执行及与Worker、Driver的信息同步.

#### Driver

Application的驱动程序,Application通过Driver与Cluster Manager、Executor进行通信.Driver可以再Application中,也可以由Application提交Cluster Manager由Cluster Manager安排Worker运行.

#### Application

用户使用Spark提供的API编写的应用程序,Application通过SparkApi进行RDD的转换和DAG的构建,并通过Driver将Application注册到Cluster Manager.Cluster manager将会根据Applicatin的资源需求,通过一级分配将Executor、内存、CPU等资源分配给Application。Driver通过二级分配将Executor等资源分配给每一个任务，Application最后通过Driver高速Executor运行任务。


## Spark的基础设施

### Spark的配置

Spark的配置通过以下三种方式获取：

 1. 来源于系统参数（即使用System.getProperties获取的属性）中以spark.作前缀的部分属性;
 2. 使用SparkConf的Api进行设置;
 3. 从其他SparkConf中克隆

### Spark内置RPC框架


![Spark内置RPC框架的基本框架](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544410602435.png)


#### RPC配置TransportConf

Spark通常使用SparkTransportConf创建TransportConf

``` scala
object SparkTransportConf {
  /**
   * Specifies an upper bound on the number of Netty threads that Spark requires by default.
   * In practice, only 2-4 cores should be required to transfer roughly 10 Gb/s, and each core
   * that we use will have an initial overhead of roughly 32 MB of off-heap memory, which comes
   * at a premium.
   *
   * Thus, this value should still retain maximum throughput and reduce wasted off-heap memory
   * allocation. It can be overridden by setting the number of serverThreads and clientThreads
   * manually in Spark's configuration.
   */
  private val MAX_DEFAULT_NETTY_THREADS = 8

  /**
   * Utility for creating a [[TransportConf]] from a [[SparkConf]].
   * @param _conf the [[SparkConf]]
   * @param module the module name
   * @param numUsableCores if nonzero, this will restrict the server and client threads to only
   *                       use the given number of cores, rather than all of the machine's cores.
   *                       This restriction will only occur if these properties are not already set.
   */
  def fromSparkConf(_conf: SparkConf, module: String, numUsableCores: Int = 0): TransportConf = {
    val conf = _conf.clone

    // Specify thread configuration based on our JVM's allocation of cores (rather than necessarily
    // assuming we have all the machine's cores).
    // NB: Only set if serverThreads/clientThreads not already set.
    val numThreads = defaultNumThreads(numUsableCores)
    conf.setIfMissing(s"spark.$module.io.serverThreads", numThreads.toString)
    conf.setIfMissing(s"spark.$module.io.clientThreads", numThreads.toString)

    new TransportConf(module, new ConfigProvider {
      override def get(name: String): String = conf.get(name)
      override def get(name: String, defaultValue: String): String = conf.get(name, defaultValue)
      override def getAll(): java.lang.Iterable[java.util.Map.Entry[String, String]] = {
        conf.getAll.toMap.asJava.entrySet()
      }
    })
  }

  /**
   * Returns the default number of threads for both the Netty client and server thread pools.
   * If numUsableCores is 0, we will use Runtime get an approximate number of available cores.
   */
  private def defaultNumThreads(numUsableCores: Int): Int = {
    val availableCores =
      if (numUsableCores > 0) numUsableCores else Runtime.getRuntime.availableProcessors()
    math.min(availableCores, MAX_DEFAULT_NETTY_THREADS)
  }
}
```

#### RPC客服端工厂TransportClientFactory


TransportClientFactory构造器的实现

``` java
 public TransportClientFactory(
      TransportContext context,
      List<TransportClientBootstrap> clientBootstraps) {
    this.context = Preconditions.checkNotNull(context);
    this.conf = context.getConf();
    this.clientBootstraps = Lists.newArrayList(Preconditions.checkNotNull(clientBootstraps));
    this.connectionPool = new ConcurrentHashMap<>();
    this.numConnectionsPerPeer = conf.numConnectionsPerPeer();
    this.rand = new Random();

    IOMode ioMode = IOMode.valueOf(conf.ioMode());
    this.socketChannelClass = NettyUtils.getClientChannelClass(ioMode);
    this.workerGroup = NettyUtils.createEventLoop(
        ioMode,
        conf.clientThreads(),
        conf.getModuleName() + "-client");
    this.pooledAllocator = NettyUtils.createPooledByteBufAllocator(
      conf.preferDirectBufs(), false /* allowCache */, conf.clientThreads());
    this.metrics = new NettyMemoryMetrics(
      this.pooledAllocator, conf.getModuleName() + "-client", conf);
  }
```
TransportClientFactory构造器中的各参数如下:

 - context: 参数传递TransportContext的引用.
 - conf:指TransportConf,这里通过调用TransportContext的getConf获取
 - clientBootstraps: 参数传递的TransportClientBootstrap列表
 - connectionPool:针对每个Socket地址的连接池ClientPool的缓存.

![ConnectionPool的数据结构](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1544412797134.png)

- numConnectionPerPeer:从TransportConf获取的key为"saprk.+模块名+.io.num-ConnectionPerPeer"的属性值.此属性值用于指定对等节点间的连接数.这里的模块名实际为TransportConf的module字段.
- rand:对Socket地址对应的连接池ClientPool中缓存的TransportClient进行随机选择,对每个连接座负载均衡.
- ioMode: IO模式
- socketChannelClass:客服端Channel被创建时使用的类,通过ioMode来匹配,默认为NioSocketChannel,Spark还支持EpollEventLoopGroup
- workerGroup:根据Netty的规范,客服端只有worker组,所以此处创建Worker-Group 。workerGroup的实际类型是NioEventLoopGroup.
- pooledAllocator：汇集ByteBuf但对线程缓存禁用的分配器。

