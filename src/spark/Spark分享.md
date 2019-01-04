---
title: Spark分享
tags: 作者:大数据应用开发
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]

## Stage

DAGScheduler会将Job的RDD划分到不同的Stage,并构建这些Stage的依赖关系.这样可以使得没有依赖关系的Stage并行执行,并保证有依赖关系的Stage顺序执行。并行执行能够有效的利用集群资源，提升运行效率，而串行则适用于那些在时间和数据资源上存在强制依赖的场景。Stage分为需要处理Shuffle的ShuffleMapStage和最下游的ResultStage。

### ResultStage

ResultStage可以使用指定的函数对RDD中的分区进行计算并得出最终结果。 ResultStage判断一个分区是否完成，是通过ActiveJob的Boolean类型数组finished，因为finished记录了每一个分区是否完成。


### ShuffleMapStage

ShuffleMapStage是DAG调度流程的中间Stage，它可以包括一到多个ShuffleMapTask，这些ShuffleMapTask将生成于Shuffle的数据。ShuffleMapStage一般是ResultStage或者是其他ShuffleMapStage的前置Stage，ShuffleMapTask则通过Shuffle与下游Stage中的Task串联起来。从ShuffleMapStage的命名可以看出，它将对Shuffle的数据映射到下游Stage的各个分区中。

### StageInfo

StageInfo用于描述Stage信息，并可以传递给Sparklistener。
## DAGScheduler

DAGSchedule实现了面向DAG的高层次调度，即将DAG中的各个RDD划分到不同的Stage。DAGSchedule可以通过计算将DAG中的一系列RDD划分到不同的Stage，然后构建这些Stage之间的父子关系，最后将每个Stage按照Partition切分为多个Task，并以Task集合（即TaskSet）的形式提交给底层的TaskScheduler。 所有的组件都通过向DAGScheduler投递DAGSchedulerEvent来使用DAGScheduler。DAGScheduler内部的DAGSchedulerEventProcessLoop将处理这些DAGSchedulerEvent，并调用DAGScheduler的不同方法。JobListener用于堆作业中每个Task执行成功或失败进行监听，JobWaiter实现了JobListener并最终确定作业的成功与失败。

### DAGSchedulerEventProcessLoop的介绍

DAGSchedulerEventProcessLoop是DAGScheduler内部的事件循环处理器，用于处理DAGSchedulerEvent类型的事件。DAGSchedulerEventProcessLoop的实现与LiveListenerBus非常的相似。

### Stage的划分与提交

![Stage的划分与提交核心方法](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546060011376.png)

### DAGScheduler的调度流程

![DAGScheduler的调度流程](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1545272317529.png)

1. 表示应用程序通过对Spark API的调用，进行一系列RDD转换构建出RDD之间的依赖关系后，调用DAGScheduler的runJob方法将RDD及其血缘关系中的所有RDD传递给DAGScheduler进行调度。
2. DAGScheduler的runJob方法实际通过调用DAGScheduler的submitJob方法向DAGSchedulerEventProcessLoop发送JobSubmitted事件。DAGSchedulerEventProcessLoop接收到JobSubmitted事件后，将JobSubmitted事件放入事件队列(eventQueue)
3. DAGSchedulerEventProcessLoop内部的轮询线程eventThread不断从事件队列中获取DAGSchedulerEvent事件，并调用DAGSchedulerEventProcessLoop的doOnReceive方法对事件进行处理。
4. DAGSchedulerEventProcessLoop的doOnReceive方法处理JobSubmitted事件时，将调用DAGScheduler的handleJobSubmitted方法。handleJobSubmitted方法将对RDD构建Stage及Stage之间的依赖关系。
5. DAGScheduler首先把最上游的Stage中的Task集合提交给TaskScheduler，然后逐步将下游的Stage中的Task集合提交给TaskScheduler。TaskScheduler将对Task集合进行调度。


## Task任务的提交


一个Spark Application分为stage级别和task级别的调度， task来源于stage，所有本文先从stage提交开始讲解task任务提交。


![架构图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572547027.png)


Standalone模式提交运行流程图：



![Standalone模式提交运行流程图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572577374.png)




![TaskSchedulerderImpl的调度流程](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546497529110.png)

 1. 代表DAGScheduler调用TaskScheduler的submitTasks方法向TaskScheduler提交TaskSet。
 2. 代表TaskScheduler接收到TaskSet后，创建对此TaskSet进行管理的TaskSetManager，并将此TaskSetManager通过调度池构造器添加到根调度池中。
 3. 代表TaskScheduler调用SchedulerBackend的reviveOffers方法给Task提供资源。
 4. SchedulerBackend向RpcEndpoint发送ReviveOffers消息。
 5. RpcEndpoint将调用TaskScheduler的resourceOffers方法给Task提供资源。
 6. TaskScheduler调用根调度池的getSortedTaskSetQueue方法对所有TaskSetManager按照调度算法进行排序后，对TaskSetmanager管理的TaskSet按照“最大本地性”的原则选择其中的Task,最后为Task创建尝试执行信息、对Task进行序列化、生成TaskDescription等。


首先写一个WordCount代码（这个代码，为了观察多个Shuffle操作，我写了两个reducebykey 函数）
源代码：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572632480.png)


直接执行代码，查看spark执行程序时，将代码划分stage生成的DAG流程图



![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572650152.png)


可知： WordCount 在stage划分的时候，划分为三个stage 
即在代码中如下标识：



![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572696563.png)


讲TaskScheduler ，先从DAGScheduler中提交任务开始吧，其中在stage划分task的时候，涉及到一些优化算法。
org.apache.spark.scheduler.DAGScheduler#handleMapStageSubmitted
这个方法主要有三个部分：
1. 创建finalStage

``` scala?linenums
finalStage = getOrCreateShuffleMapStage(dependency, jobId)
```

2. 创建ActiveJob

``` scilab?linenums
val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
```

3. 提交stage

``` scala?linenums
submitStage(finalStage)
```


直接看第三步 submitStage


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572819448.png)




这个是提交stage方法。
里面是一个递归方法，举例：
在代码中， 划分为三个stage：
stage0  ---> stage1     ---> stage2
 submitStage(stage: Stage) 这个方法先传入的是 finalStage（stage2）
在方法里面循环递归， 分别寻找stage的父stage， 即 stage2 找到 stage1 ， stage1找到stage0
stage0 没有父stage 即走 提交方法：
submitMissingTasks(stage: Stage, jobId: Int)
好，接下来，我们看submitMissingTasks
可以看到入参： ShuffleMapStage 0 和 jobId 0
 找出当前stage的所有分区中，还没计算完分区的stage
 
 
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572869602.png)
 
 
 
 ShuffleMapStage
stage.findMissingPartitions获取需要计算的分区，不同的stage有不同的实现：


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572896923.png)



ResultStage

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572923068.png)



计算 分区的最佳位置 ：  taskIdToLocations

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572943576.png)

计算最佳位置的核心方法： getPreferredLocsInternal  (递归方法)



![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572963275.png)


这个开始传入的RDD：3，
rdd：3找不到最佳位置， 找到rdd：3的父级rdd：2，
rdd2，找不到最佳位置，找到rdd2的父级rdd1
rdd1有最佳位置，直接返回： 具体的机器地址：


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546572994482.png)


广播信息：


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573010555.png)



为每一个MapStage的分区 创建一个 ShuffleMapTask 或者 ResultTask 

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573025946.png)


将ShuffleMapTask 或者 ResultTask  封装成taskSet，提交Task


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573052486.png)



在这里执行的是

``` scala?linenums
taskScheduler.submitTasks(new TaskSet(
  tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
```

接着调用执行的是：

``` scala?linenums
org.apache.spark.scheduler.TaskSchedulerImpl#submitTasks
```


这个方法一共干了两件事:
1. 创建TaskSetManager
2. 资源调度&运行task
具体详情请参考注解：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573414898.png)

直接看
backend.reviveOffers()

backend 为： 

YarnClusterSchedulerBackend  ==继承=》 YarnSchedulerBackend     ==继承=》 CoarseGrainedSchedulerBackend

所以这个方法执行的是CoarseGrainedSchedulerBackend 中的reviveOffers 方法：


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573438065.png)


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573445201.png)



最终走的是 makeOffers 这个方法

为所有的executor 提供虚拟的资源。。。。。。



![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573462936.png)


## Task资源调度

TaskSchedulerImpl#resourceOffers
这个方法有点长，具体如注解

被集群manager调用以提供slaves上的资源。我们通过按照优先顺序询问活动task集中的task来回应。
我们通过循环的方式将task调度到每个节点上以便tasks在集群中可以保持大致的均衡。

![](https://markdown.xiaoshujiang.com/img/spinner.gif "[[[1546573628903]]]" )


## Task启动