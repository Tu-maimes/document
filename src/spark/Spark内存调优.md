---
title: Spark内存调优 
tags: spark
grammar_cjkRuby: true
---


### Spark 内存调优以及 JVM 调优
目前Spark使用的内存管理模型有两个,分别是:

 1. StaticMemoryManager
 2. UnifiedMemoryManager

而StaticMemoryManager是1.6之前的版本使用的内存管理模型.UnifiedMemoryManager是1.6之后使用的内存管理模型.
在SparkEvn中,通过spark.memory.useLegacyMode参数来进行内存管理模型的配置,默认false,即默认使用UnifiedMemoryManager.
```scala
val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
    val memoryManager: MemoryManager =
      if (useLegacyMemoryManager) {
        new StaticMemoryManager(conf, numUsableCores)
      } else {
        UnifiedMemoryManager(conf, numUsableCores)
      }
```

#### StaticMemoryManager
StaticMemoryManager模式下,分配给Spark的内存主要分为3部分.

 1. Storage 部分,主要用于存储 RDD 缓存的数据和 broadcast 广播的数据,这部分内存占给定 spark 内存的60%.其中这部分的10%用于Storage 内存的预留内存.剩下的90%用于存储.而剩下的90%又划分出20%的大小单独用于缓存 iterator 形式的 Block 数据,即 unroll 部分.
 2. Shuffle 部分,这部分内存占给定 spark 内存的20%,主要用于存储 Shuffle 过程中的中间数据(或者可以说用于 Task 任务的计算),这部分也有20%的大小用于预留内存,80%的大小用于真正的计算用.
 3. Other 部分,这部分内存占给定 spark 内存的20%,用于存储用户定义的数据结构或者 spark 内部的元数据.

StaticMemoryManager模式下,spark 各部分内存的分配图:
![spark 各部分内存的分配图](https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1554637667811.png)

#### UnifiedMemoryManager
<!-- https://0x0fff.com/spark-memory-management/#comment-1188 -->
UnifiedMemoryManager模式下,spark 内存主要分为三块:

 1. Storage&Execution,其中 Storage 主要用于存储 RDD 缓存的数据和 broadcast 广播的数据.Execution主要用于 Task 的计算(存储 shuffle 中间数据).Storage 和 Execution 大小默认比例为50%,但是,这是个软边界限制,也即当一方不足时,可以申请向另一方借用.Storage&Execution由spark.memory.fraction参数控制,默认60%.Storage与 Execution 占比由spark.memory.storageFraction参数控制,默认各占50%.
 2. User Memory,用户内存区域,主要用于存储 spark 的 RDD 转换过程中使用的数据结构,官网上说的是 Spark完全没有考虑你在那里做什么以及你是否尊重这个边界.这部分内存的大小就是 spark 分配给用于存储和计算的内存之外的内存.默认为(1-spark.memory.fraction)
 3. Reserved Memory,系统保留内存,不在任何生产环境下使用.这块区域的大小是默认值300M,不能更改.

UnifiedMemoryManager模式下,spark 各部分内存的分配图:
![UnifiedMemoryManager模式下,spark 各部分内存的分配图](https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1554645426140.png)

spark 内存调优参数:

|   参数  |  默认值   |   作用  |
| --- | --- | --- |
|   spark.memory.useLegacyMode  |   false   |   选用内存管理模式,默认为 false,即默认使用 UnifiedMemoryManager,为 true 时,使用StaticMemoryManager  |
|   spark.storage.safetyFraction   | 0.9 |StaticMemoryManager 模式下,实际用于存储的安全系数|
|   spark.storage.memoryFraction   |  0.6   |  StaticMemoryManager 模式下,用于缓存 RDD 数据以及 broadcast 数据,实际大小为`systemMaxMemory*0.6*0.9*(1-unrollFraction)`  |
|   spark.shuffle.safetyFraction   |   0.8   |   StaticMemoryManager 模式下实际用于计算的安全系数   |
|   spark.shuffle.memoryFraction   |   0.2  |   StaticMemoryManager 模式下,用于存储 shuffle 中间数据(task 计算) 实际大小为 `systemMaxMemory * 0.2*0.8`|
|   spark.storage.unrollFraction    |	0.2	|	StaticMemoryManager 模式下,unroll 内存所占比例,主要用于缓存 iterator 形式的 Block 数据,实际大小为`systemMaxMemory*0.6*0.9*unrollFraction` 	|
|	spark.memory.fraction	|	0.6	|	控制UnifiedMemoryManager模式下,用于存储和计算的总内存,实际大小为`(systemMemory-300M)*0.6`	|
|	spark.memory.storageFraction	|	0.5	|	控制UnifiedMemoryManager模式下,用于存储和计算的内存占比,实际大小为`(systemMemory-300M)*0.6*0.5`.这是个软边界,当一方不足时,可以向另一方借用,比如 storage 不足时,可以向 execution 借用	|
|	spark.memory.offHeap.enabled	|	false	|	是否使用堆外内存,默认是不使用堆外内存的,当开启的时候,需要指定堆外内存的大小并且大于0  <!--同时还要满足三个条件:1.Serializer支持relocation  2.无map端combine操作 3.reduce端partition个数小于2的24次方(16777216) -->	|
|	spark.memory.offHeap.size	|	0	|	堆外内存大小,当启用堆外内存时,这个参数的值需要大于0,单位字节	|
|	spark.storage.replication.proactive	|	false	|	默认是不启用该参数的,如果启用,则会在Executor故障丢失了rdd时,使用 RDD 的副本进行恢复,而不用重新开始计算. 如果开启,将会使用额外的存储空间.	|
|	spark.cleaner.periodicGC.interval	|	30min	|	控制触发垃圾回收的频率	|
|	spark.cleaner.referenceTracking	|	true	|	是否启用上下文清理.清理那些超出应用范围的 RDD,Shuffle 对应的 map 任务状态,Shuffle 元数据,Broadcast 对应以及 RDD 的 Checkpoint 数据	|
|	spark.cleaner.referenceTracking.blocking	|	true	|	清理非Shuffle的其它数据是否是阻塞式的	|
|	spark.cleaner.referenceTracking.blocking.shuffle	|	false	|	清理Shuffle数据是否是阻塞式的,清理MapOutputTracker中指定ShuffleId对应的map任务状态和ShuffleManager中注册的ShuffleId对应的Shuffle元数据。	|
|	spark.driver.extraJavaOptions	|	-	|	用于设置 driver 端的 jvm 参数	|
|	spark.executor.extraJavaOptions	|	-	|	用于设置 executor 端的 jvm 参数	|



#### JVM 参数调优

步骤:
1. 在 添加打印 GC 日志的参数:`spark.executor.extraJavaOptions=-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps`
2. 提交 spark 应用,查看花费时间最多的 stage 中,GC 花费时间最大的 task 任务的日志.
3. 查看 GC 日志:
3.1 如果一个task 任务完成之前发生了多次FullGC,说明没有分配足够多的内存,如果机器内存还足够的话,可以增加executor-memory 参数的值.同时,在代码中酌情较少 RDD 的缓存.在一个,jvm 的垃圾收集器使用更高效的 G1收集器
3.2 增加年轻代的堆内存空间,这样可以减少年轻代内存区域发生GC的次数,从而减少本不应该到老年代的对象的GC 年龄,这样可以减少进入老年代的对象,从而减少老年代发生 fullGC 的频率.设置新生代堆大小的参数:-Xmn,设置时,可以根据 Eden区的大小进行设置,大概为 Eden 区大小的4/3倍.
3.3 如果源数据是从HDFS读取，则可以使用从HDFS读取的数据块的大小来估计任务使用的内存量。请注意，解压缩块的大小通常是块大小的2或3倍。因此，如果我们希望有3或4个任务的工作空间，并且HDFS块大小为128 MB，我们可以估计Eden的大小`4*3*128`MB,则新生代的大小为`4*3*128*4/3`MB 
<!-- 
http://spark.apache.org/docs/2.3.2/tuning.html#memory-management-overview
 -XX:+UseG1GC ：打开G1收集器开关
        -XX:MaxGCPauseMillis ：指定最大停顿时间。如果任何一次停顿超过这个值，G1就会尝试调整新生代和老年代的比例、调整对大小、调整晋升年龄等，试图达到预设目标。
        -XX:ParallelGCThreads ：设置GC线程数量
        -XX:InitiatingHeapOccupancyPercent ：指定当整个堆使用率达到多少时，触发并发标记周期的执行，默认值是45%。
-->

#### 内存调优建议
 1. 使用高效的数据序列化类KryoSerializer
	 - 参数:spark.serializer:org.apache.spark.serializer.KryoSerializer
 2. 设计数据结构时优先使用对象数组和基本类型,使用到集合类库时,可以使用 [fastutil](http://fastutil.di.unimi.it/) 库.
 3. 合理的使用内存管理以及内存管理参数设置
	 1. 内存管理模型选择:
		<!-- - 如果是你的计算比较复杂的情况，使用新型的内存管理 (UnifiedMemoryManager) 会取得更好的效率，但是如果说计算的业务逻辑需要更大的缓存空间，此时使用老版本的固定内存管理 (StaticMemoryManager) 效果会更好。-->
		1. 如果我们的spark 应用计算比较复杂,没法准确的预估计算和存储所占的比例,这时我们可以依赖UnifiedMemoryManager管理模型计算和存储软边界的特性选择使用UnifiedMemoryManager,这也是spark1.6之后系统默认的.
		2. 而如果我们 spark 应用计算业务逻辑时需要更大的存储空间,比如代码逻辑中使用频繁的使用广播变量,或者RDD 的 血统链比较长,需要使用 rdd 的 persist()等操作,这种情况下可以选择使用StaticMemoryManager的固定内存管理模型比较好,并且需要适当的spark.storage.memoryFraction参数系数.
	 2. 在使用StaticMemoryManager管理内存时,如果 spark 应用用到了大量的广播或者 RDD 缓存,可以增加spark.storage.memoryFraction参数的值.反之,如果计算比较复杂,计算的过程中很少将 RDD 进行缓存,可以减小spark.storage.memoryFraction的值,同时增加spark.shuffle.memoryFraction的值.
	 3. 在使用UnifiedMemoryManager管理内存时,如果计算比较复杂,可以将参数spark.memory.fraction稍微调大,同时调整spark.memory.storageFraction的值,默认是0.5,可以根据实际情况调整为如0.4(虽然在计算的过程中可以进行相互借用)
 4. jvm 参数优化
 <!-- 
 yarn下：

--conf  spark.yarn.executor.memoryOverhead=2048 单位M

standalone下：

--conf  spark.executor.memoryOverhead=2048单位M
-->