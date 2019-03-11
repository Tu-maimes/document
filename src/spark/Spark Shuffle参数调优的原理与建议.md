---
title: Spark Shuffle参数调优的原理与建议
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]

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



### Spark配置参数的源码详解(Spark2.3)

#### spark.shuffle.manager 

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552274797735.png)

该参数用于设置ShuffleManager的类型。Tungsten-Sort与Sort类似，但是使用了Tungsten计划中的堆外内存管理机制，内存使用效率更高。从源码中可用看到目前版本只有Sort与Tungsten-Sort两个选项可以选择，曾经的Hash-Based Shuffle算法在新版本中已经被废弃。SortShuffleManager默认对数据进行排序，因此如果用户的业务逻辑中没有排序可以通过设置`spark.shuffle.sort.bypassMergeThreshold`参数来避免不必要的排序操作，同时提供较好的磁盘读写性能。

#### spark.reducer.maxReqsInFlight与spark.reducer.maxBlocksInFlightPerAddress

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552280842089.png)

``` scala?linenums
 private[this] def splitLocalRemoteBlocks(): ArrayBuffer[FetchRequest] = {
    // 远程请求长度不超过maxBytesInFlight / 5;
    // 让它们小于maxBytesInFlight的原因是允许从最多5个节点进行多个并行获取，而不是阻塞从一个节点读取输出.
    val targetRequestSize = math.max(maxBytesInFlight / 5, 1L)
    logDebug("最大飞行的字节数: " + maxBytesInFlight + ", 目标要求的大小: " + targetRequestSize
      + ", maxBlocksInFlightPerAddress: " + maxBlocksInFlightPerAddress)

    // 分割本地块和远程块。为了限制飞行中的数据量，将远程块进一步划分为最多maxBytesInFlight大小的fetchrequest.
    val remoteRequests = new ArrayBuffer[FetchRequest]

    // 轨道总块数(包括零大小块)
    var totalBlocks = 0
    for ((address, blockInfos) <- blocksByAddress) {
      totalBlocks += blockInfos.size
      if (address.executorId == blockManager.blockManagerId.executorId) {
        // 过滤掉零大小的块
        localBlocks ++= blockInfos.filter(_._2 != 0).map(_._1)
        numBlocksToFetch += localBlocks.size
      } else {
        val iterator = blockInfos.iterator
        var curRequestSize = 0L
        var curBlocks = new ArrayBuffer[(BlockId, Long)]
        while (iterator.hasNext) {
          val (blockId, size) = iterator.next()
          // 跳过空块
          if (size > 0) {
            curBlocks += ((blockId, size))
            remoteBlocks += blockId
            numBlocksToFetch += 1
            curRequestSize += size
          } else if (size < 0) {
            throw new BlockException(blockId, "消极的块大小 " + size)
          }
          //  满足当中的任意一个条件就创建一个新的请求
          if (curRequestSize >= targetRequestSize ||
              curBlocks.size >= maxBlocksInFlightPerAddress) {
            // 添加这个获取请求
            remoteRequests += new FetchRequest(address, curBlocks)
            logDebug(s"创建获取请求的 $curRequestSize at $address "
              + s"with ${curBlocks.size} blocks")
            curBlocks = new ArrayBuffer[(BlockId, Long)]
            curRequestSize = 0
          }
        }
        // 添加最后的请求
        if (curBlocks.nonEmpty) {
          remoteRequests += new FetchRequest(address, curBlocks)
        }
      }
    }
    logInfo(s"从$totalBlocks 块中获取 $numBlocksToFetch 非空块")
    remoteRequests
  }
```

#### spark.maxRemoteBlockSizeFetchToMem

![spark.maxRemoteBlockSizeFetchToMem](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552282855711.png)


#### spark.shuffle.compress

``` scala?linenums
 /**
     * 合并零个或多个泄漏文件在一起，选择最快的合并策略基于泄漏的数量和IO压缩编解码器.
     *
     * @return 合并文件中的分区长度。
     */
    private long[] mergeSpills(SpillInfo[] spills, File outputFile) throws IOException {
        //  是否开启shuffle的压缩机制
        final boolean compressionEnabled = sparkConf.getBoolean("spark.shuffle.compress", true);
        // 解压缩的编码器
        final CompressionCodec compressionCodec = CompressionCodec$.MODULE$.createCodec(sparkConf);
        // 快速合并启用机制
        final boolean fastMergeEnabled =
                sparkConf.getBoolean("spark.shuffle.unsafe.fastMergeEnabled", true);
        // 支持快速合并
        final boolean fastMergeIsSupported = !compressionEnabled ||
                CompressionCodec$.MODULE$.supportsConcatenationOfSerializedStreams(compressionCodec);
        // 是否启用加密
        final boolean encryptionEnabled = blockManager.serializerManager().encryptionEnabled();
        try {
            if (spills.length == 0) {
                new FileOutputStream(outputFile).close(); // 创建一个空文件
                return new long[partitioner.numPartitions()];
            } else if (spills.length == 1) {
                // 在这里，我们不需要执行任何指标更新，因为写入这个输出文件的字节已经被计算为写入的随机字节.
                Files.move(spills[0].file, outputFile);
                return spills[0].partitionLengths;
            } else {
                final long[] partitionLengths;
                // There are multiple spills to merge, so none of these spill files' lengths were counted
                // towards our shuffle write count or shuffle write time. If we use the slow merge path,
                // then the final output file's size won't necessarily be equal to the sum of the spill
                // files' sizes. To guard against this case, we look at the output file's actual size when
                // computing shuffle bytes written.
                //有多个溢出要合并，所以这些溢出文件的长度都没有计入我们的随机写入计数或随机写入时间。
                // 如果我们使用缓慢的合并路径，那么最终输出文件的大小不一定等于溢出文件大小的总和。
                // 为了防止这种情况，我们在计算写入的随机字节时查看输出文件的实际大小
                // We allow the individual merge methods to report their own IO times since different merge
                // strategies use different IO techniques.  We count IO during merge towards the shuffle
                // shuffle write time, which appears to be consistent with the "not bypassing merge-sort"
                // branch in ExternalSorter.
                //由于不同的合并策略使用不同的IO技术，我们允许各个合并方法报告它们自己的IO时间。
                // 我们将合并过程中的IO计算到shuffle写入时间中，这似乎与ExternalSorter中的“不绕过合并排序”分支一致。

                // 启用快速合并并且支持快速合并
                if (fastMergeEnabled && fastMergeIsSupported) {
                    // 两种不同的合并策略


                    // 压缩被禁用，或者使用我们正在使用IO压缩编解码器，该编解码器支持对串联的压缩流进行解压，
                    // 因此我们可以执行快速溢出合并，而不需要解释溢出的字节.
                    // 启用NIO模式不采用加密
                    if (transferToEnabled && !encryptionEnabled) {
                        logger.debug("使用基于传输的快速合并");
                        partitionLengths = mergeSpillsWithTransferTo(spills, outputFile);
                    } else {
                        logger.debug("使用基于文件流的快速合并");
                        partitionLengths = mergeSpillsWithFileStream(spills, outputFile, null);
                    }
                } else {
                    logger.debug("使用慢合并");
                    partitionLengths = mergeSpillsWithFileStream(spills, outputFile, compressionCodec);
                }
                // When closing an UnsafeShuffleExternalSorter that has already spilled once but also has
                // in-memory records, we write out the in-memory records to a file but do not count that
                // final write as bytes spilled (instead, it's accounted as shuffle write). The merge needs
                // to be counted as shuffle write, but this will lead to double-counting of the final
                // SpillInfo's bytes.
                //当关闭一个已经溢出一次但也有内存中的记录的UnsafeShuffleExternalSorter时，
                // 我们将内存中的记录写入文件，但不将最终的写入计算为溢出字节(相反，它被认为是随机写入)。
                // 合并需要作为随机写入进行计数，但这将导致对最后sp伊利诺伊州fo字节的重复计数
                writeMetrics.decBytesWritten(spills[spills.length - 1].file.length());
                writeMetrics.incBytesWritten(outputFile.length());
                return partitionLengths;
            }
        } catch (IOException e) {
            if (outputFile.exists() && !outputFile.delete()) {
                logger.error("Unable to delete output file {}", outputFile.getPath());
            }
            throw e;
        }
    }
```

#### spark.io.compression.codec

spark.io.compression.codec参数用来压缩内部数据，如：RDD分区、广播变量和Shuffle输出等数据。从如下源码中可知默认的压缩方式是lz4

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552284233060.png)

#### spark.shuffle.file.buffer

在ShuffleMapTask端通常也会增大Map任务的写磁盘的缓存，默认是32KB。Spark.Shuffle.file.buffler参数用于设置Shuffle write Task的BufferedOutputStream的Buffer缓冲大小。将数据写入磁盘文件之前，先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。可以视集群资源来提高此参数，从而减少Shuffle Writer 过程中溢写磁盘文件的次数，也就减少磁盘IO次数，进而提升性能。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552286322444.png)


#### spark.shuffle.io.maxRetries与 spark.shuffle.io.retryWait


Shuffle Read Task从Shuffle Write Task 所在节点拉取属于自己的数据时，因某种网络异常导致拉取失败，是会自动进行重试的。如果在指定的次数内拉取还是没有成功，就可能导致作业执行失败。
调优建议：对于那些包含了特别耗时的Shuffle操作时，建议增加最大的重试次数，以避免由于JVM的Full GC或者网络不稳定等因素导致的数据拉取失败。对于超大的数据量时可以提升集群的稳定性。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552287095270.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552287142481.png)


#### spark.shuffle.io.numConnectionsPerPeer

重新使用主机之间的连接，以减少大型集群的连接建立，对于具有多个硬盘和少量主机的集群，这可能导致并发性不足，以使所有磁盘饱和。此参数用于获取数据的两个节点之间的并发连接数。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1552287764826.png)


#### spark.shuffle.io.preferDirectBufs

堆外缓存可以有效减少垃圾回收和缓存复制。对于堆外内存紧张的用户来说，可以考虑禁用这个选项。以迫使所有孽畜都分配在堆上。


