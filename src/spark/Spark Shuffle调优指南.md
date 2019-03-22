---
title: Spark Shuffle调优指南
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc]

## Shuffle对性能消耗的原理详解

### Spark Shuffle过程中影响性能的操作：

 1. 磁盘I/O
 2. 网络I/O
 3. 压缩
 4. 解压缩
 5. 序列化
 6. 反序列化

调优是一个动态的过程，需要根据业务数据的特性和硬件设备来综合调优。

[Spark的配置参数官网链接](http://spark.apache.org/docs/latest/configuration.html)

|属性参数|默认值|含义|
|---|---|---|
|==spark.reducer.maxSizeInFlight #F44336==|48MB|由于每个输出都需要创建一个缓冲区来接收它，因此每个reduce任务的内存开销都是固定的，所以要保持较小的内存，除非您有大量的内存|
|==spark.reducer.maxReqsInFlight==|Int.MaxValue|这种配置限制了在任何给定点获取块的远程请求的数量。当集群中的主机数量增加时，可能会导致到一个或多个节点的大量入站连接，从而导致工作人员在负载下失败。通过允许它限制获取请求的数量，可以缓解这种情况。|
|==spark.reducer.maxBlocksInFlightPerAddress==|Int.MaxValue|这种配置限制了从给定主机端口为每个reduce任务获取的远程块的数量。当一次获取或同时从给定地址请求大量块时，可能会导致服务执行器或节点管理器崩溃。当启用外部洗牌时，这对于减少节点管理器上的负载特别有用。您可以通过将其设置为一个较低的值来缓解这个问题。|
|==spark.maxRemoteBlockSizeFetchToMem==|Long.MaxValue|当块的大小(以字节为单位)超过这个阈值时，远程块将被取到磁盘。这是为了避免占用太多内存的巨大请求。默认情况下，这只对块大于 2GB启用，因为这些块不能直接获取到内存中，无论有什么资源可用。但是它可以被降低到一个更低的值。为了避免在较小的块上使用太多的内存。注意，此配置将同时影响shuffle获取和块管理器远程块获取。对于启用外部洗牌服务的用户，此功能只能在外部洗牌服务比Spark 2.2更新时使用。|
|==spark.shuffle.compress #F44336==|true|是否压缩map输出文件。压缩将使用 spark.io.compression.codec。|
|==spark.shuffle.file.buffer #F44336==|32KB|每个shuffle文件输出流的内存缓冲区大小。这些缓冲区减少了在创建中间shuffle文件时进行的磁盘搜索和系统调用的次数。|
|==spark.shuffle.io.maxRetries #9C27B0==|3|（仅限Netty）如果将此设置为非零值，在IO相关异常导致获取数据失败，将自动重试。重试将有助于保障长时间GC停顿或瞬时网络连接问题情况下Shuffle的稳定性|
|==spark.shuffle.io.numConnectionsPerPeer==|1|（仅限Netty）重新使用主机之间的连接，以减少大型群集的连接建立。对于具有许多硬盘和少量主机的集群，这可能导致并发性不足以使所有磁盘饱和，因此用户可能会考虑增加此值。|
|==spark.shuffle.io.preferDirectBufs==|ture|（仅限Netty）非堆缓冲区用于在Shuffle和缓存块传输过程中减少垃圾回收。对于非堆内存严格限制的环境，用户可能希望将其关闭，以强制Netty的所有分配都在堆上|
|==spark.shuffle.io.retryWait #9C27B0==| 5S|（仅限Netty）在重试提取之间等待多长时间。默认情况下，重试导致的最大延迟为15秒，计算方式为maxRetries * retryWait。|
|==spark.shuffle.service.enabled #FF9800==|false|启用外部Shuffle服务。此服务保留由Executor写入的Shuffle文件，以便Executors可以安全地删除。如果spark.dynamicAllocation.enabled为“true”，则必须启用此选项 。必须设置外部Shuffle服务才能启用它。|
|spark.shuffle.service.port|7337|外部Shuffle服务运行的端口|
|==spark.shuffle.service.index.cache.size #F44336==|100M|缓存Shuffle服务的索引文件的内存大小|
|spark.shuffle.maxChunksBeingTransferred|Long.MAX_VALUE|允许在Shuffle服务上同时传输的最大块数。请注意，当达到最大数量时，将关闭新的传入连接。客户端将根据shuffle重试配置重试（请参阅spark.shuffle.io.maxRetries和 spark.shuffle.io.retryWait），如果达到这些限制，任务将因提取失败而失败。|
|==spark.shuffle.sort.bypassMergeThreshold #F44336==|200|（高级）在基于排序的shuffle manager中，如果没有map端聚合，并且最多有这么多reduce分区，则避免合并排序数据|
|==spark.shuffle.spill.compress #F44336==|true|是否压缩在Shuffle期间溢出的数据。压缩将使用 spark.io.compression.codec。|
|spark.shuffle.accurateBlockThreshold|100 * 1024 * 1024|以字节为单位的阈值，高于该阈值可准确记录HighlyCompressedMapStatus中Shuffle块的大小。这有助于通过避免在获取shuffle块时低估shuffle块大小来防止OOM。|
|spark.shuffle.registration.timeout|5000|超时(以毫秒为单位)，用于注册到外部洗牌服务|
|spark.shuffle.registration.maxAttempts|3|当我们未能注册到外部shuffle服务时，我们将重试maxAttempts次。|





### Spark 压缩算法的比较

[压缩算法的比较](https://infoq.cn/article/2015/01/zstandard-compression-algorithm)


https://blog.csdn.net/zhuiqiuuuu/article/details/78130382


## Shuffle调优指南

### 系统架构无法避免Shuffle

 1. 本质：分布式网络应用无法避免Shuffle。
 2. Spark集群规模
 3. IO模型   异步    同步
 4. 线程模型       具有能够支持即时处理的线程
 5. Shuffle关键点       Shuffle应用本身类型         分布式网络通信

### 序列化

 1. 序列化要小    
 2. 反序列化要快
 3. 序列化是资源消耗     序列化的消耗    网络资源的消耗

### 底层释放能力

 1. 操作系统
 2. 文件句柄数
 3. 网卡特性
 4. linux内核
 5. TCP网络协议

### JVM层

 1. 控制资源

[Spark Shuffle中JVM内存使用及配置内幕详情](https://www.cnblogs.com/jcchoiling/p/6494652.html)

 2. 控制SHuffle资源有两种情况分为StaticMemoryManagement与UnifiedMemoryManager。在Spark2.0以后版本中默认使用的是UnifiedMemoryManager。

- StaticMemoryManagement

采用静态内存管理的策略时，Spark会定义一个安全空间，在默认情况下只会使用Java堆上的90%作为安全空间，在单个Executor的角度来讲，就是Heap Size x 90% (如果你的内存空间Heap非常大的话，可以尝试调高为95%)，在安全空间中也会划分三个不同的空间：一个是Storage空间、一个是Unroll空间和一个是Shuffle空间。

安全空间()：计算公式 spark.executor.memory  x  spark.storage.safetyFraction 。 也就是Heap Size x 90% 。

缓存空间(Storage) : 计算公式是 spark.executor.memory x spark.storage.safetyFraction x spark.storage.memoryFraction 。也就是Heap Size x  90% x 60% ； Heap Size x 54% 。一个应用程序可以缓存多少数据是由 safetyFraction 和 memoryFraction 这两个参数共同决定的。

Unroll空间：计算公式是spark.executor.memory x spark.storage.safetyFraction x spark.storage.memoryFraction x spark.storage.unrollFraction 。 也就是Heap Size x 90% x 60% x 20%；Heap Size x 10.8% 。你可能把数据序列化后的数据放在内存中，当你使用数据时，你需要把序列化的数据进行反序列化。
对cache缓存数据的影响是由于Unroll 是一个优先级较高的操作，进行Unroll操作的时候会占用cache的空间，而且又可以挤掉缓存在内存中的数据(如果该数据的缓存级别是MEMORY_ONLY 的话，数据可能会丢失)。

Shuffle空间 ： 计算公式是 spark.executor.memory x spark.shuffle.memoryFraction x spark.shuffle.safteyFraction 。 在Shuffle空间中也会默认80%的安全空间比例，所以应该是Heap Size x 20% x 80%；Heap Size x 16% 。 从内存的角度讲，你需要从远程抓取数据，抓取数据是一个Shuffle的过程，比如说你需要对数据进行排序，显现在这个过程中需要内存空间。

- UnifiedMemoryManager

Spark2.0以后新型 JVM Heap 分成三个部分：Reserved Memory 、User
 Memory 、Spark Memory。



spark.shuffle.manager
spark.shuffle.io.preferDirectBufs

### Spark层面

 1. Spark应用本身能力
 2. 序列化

序列化在分布式系统中扮演着重要的角色，优化Spark程序时，首当其冲的就是对序列化方式的优化。Spark为使用者提供两种序列化方式：
Java serialization: 默认的序列化方式。
Kryo serialization: 相较于 Java serialization 的方式，速度更快，空间占用更小，但并不支持所有的序列化格式，同时使用的时候需要注册class。spark-sql中默认使用的是kyro的序列化方式。
 配置：
  可以在Saprk-default.conf设置全局参数，也可以在代码的初始化时对SaprkConf设置。该参数会同时作用于机器之间数据的shuffle操作以及序列化rdd到磁盘、内存。
  Saprk不将Kyro设置成默认的序列化方式是因为需要对类进行注册，官方强烈建议在一些网络数据传输很大的应用中使用kyro序列化。

``` scala?linenums
val conf = new SparkConf()
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
conf.set("spark.kryo.registrationRequired", "true")
conf.registerKryoClasses(Array(classOf[MyClass1],classOf[MyClass2]))
val sc = new SparkContext(conf)
```
如果你需要系列化的对象比较大，可以增加参数spark.kryoserializer.buffer所设置的值。

如果你没有注册需要序列化的class，Kyro依然可以正常工作，但会但会存储每个对象的全类名(full class name),这样的使用方式往往比默认的 Java serialization 还要浪费更多的空间。
可以设置spark.kryo.registrationRequired 参数为 true，使用kyro时如果在应用中有类没有进行注册则会报错，通过报错把对应的类进行注册。
  

 3. 压缩

spark.shuffle.compress
spark.shuffle.spill.compress 
spark.shuffle.accurateBlockThreshold

spark.io.compression.codec
 
 官方提供了四种压缩方式：
|压缩方式|默认值|类名全路径|
|---|---|---|
|lz4|true|org.apache.spark.io.LZ4CompressionCodec|
|lzf|false|org.apache.spark.io.LZFCompressionCodec|
|snappy|false|org.apache.spark.io.SnappyCompressionCodec|
|zstd|false|org.apache.spark.io.ZStdCompressionCodec|

具体调优就是CPU和压缩率的权衡取舍以及网络、磁盘的能力和负载。对于RDD Cache的场合来说，绝大多数场合都是内存操作或者本地操作，所以CPU负载问题可能比IO问题更加突出。
 

以下三个参数的默认值均为32K，在压缩时用到的块的大小，降低这个块的大小也会降低Shuffle内存的使用率。
spark.io.compression.lz4.blockSize
spark.io.compression.snappy.blockSize
spark.io.compression.zstd.bufferSize


 Spark在通信层面主要控制的参数

 1. spark.reducer.maxSizeInFlight
 2. spark.reducer.maxReqsInFlight 
 3. spark.reducer.maxBlocksInFlightPerAddress
 4. spark.maxRemoteBlockSizeFetchToMem
 5. spark.shuffle.io.numConnectionsPerPeer
 6. spark.shuffle.io.maxRetries
 7. spark.shuffle.io.retryWait
 8. spark.shuffle.registration.timeout
 9. spark.shuffle.registration.maxAttempts
 10. spark.shuffle.maxChunksBeingTransferred
 11. spark.shuffle.service.enabled 
 
 5. 控制如何使用Netty



