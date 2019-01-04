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

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546573628903.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578110575.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578147694.png)



主体流程。如下：
        1、设置标志位newExecAvail为false，这个标志位是在新的slave被添加时被设置的一个标志，下面在计算任务的本地性规则时会用到；
        2、循环offers，WorkerOffer为包含executorId、host、cores的结构体，代表集群中的可用executor资源：
            2.1、更新executorIdToHost，executorIdToHost为利用HashMap存储executorId->host映射的集合；
            2.2、如果新的slave加入：
                2.2.1、executorsByHost中添加一条记录，key为host，value为new HashSet[String]()；
                2.2.2、发送一个ExecutorAdded事件，并由DAGScheduler的handleExecutorAdded()方法处理；
                2.2.3、新的slave加入时，标志位newExecAvail设置为true；
            2.3、更新hostsByRack；
        3、随机shuffle offers（集群中可用executor资源）以避免总是把任务放在同一组workers上执行；
        4、构造一个task列表，以分配到每个worker，针对每个executor按照其上的cores数目构造一个cores数目大小的ArrayBuffer，实现最大程度并行化；
        5、获取可以使用的cpu资源availableCpus；
        6、调用Pool.getSortedTaskSetQueue()方法获得排序好的task集合，即sortedTaskSets；
        7、循环sortedTaskSets中每个taskSet：
               7.1、如果存在新加入的slave，则调用taskSet的executorAdded()方法，动态调整位置策略级别，这么做很容易理解，新的slave节点加入了，那么随之而来的是数据有可能存在于它上面，那么这时我们就需要重新调整任务本地性规则；
        8、循环sortedTaskSets，按照位置本地性规则调度每个TaskSet，最大化实现任务的本地性：
              8.1、对每个taskSet，调用resourceOfferSingleTaskSet()方法进行任务集调度；
        9、设置标志位hasLaunchedTask，并返回tasks。


接下来我们重点分析一下 一些重要的代码：

返回排序过的TaskSet队列，有FIFO及Fair两种排序规则，默认为FIFO，可通过配置修改

val sortedTaskSets = rootPool.getSortedTaskSetQueue

schedulableQueue为Pool中的一个调度队列，里面存储的是TaskSetManager
在TaskScheduler的submitTasks()方法中，通过层层调用，最终通过Pool的addSchedulable()方法将之前生成的TaskSetManager加入到schedulableQueue中
而TaskSetManager包含具体的tasks
taskSetSchedulingAlgorithm为调度算法，包括FIFO和FAIR两种
这里针对调度队列， 按照调度算法对其排序， 生成一个序列sortedSchedulableQueue，

FIFO： 先入先出
FAIR： 公平调度


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578192030.png)


首先，创建一个ArrayBuffer，用来存储TaskSetManager，然后，对Pool中已经存储好的TaskSetManager，即schedulableQueue队列，按照taskSetSchedulingAlgorithm调度规则或算法来排序，得到sortedSchedulableQueue，并循环其内的TaskSetManager，通过其getSortedTaskSetQueue()方法来填充sortedTaskSetQueue，最后返回。TaskSetManager的getSortedTaskSetQueue()方法也很简单，追加ArrayBuffer[TaskSetManager]即可，如下：
我们着重来讲解下这个调度准则或算法taskSetSchedulingAlgorithm，其定义如下：
FIFO:
//  FIFO排序类中的比较函数的实现很简单：
//  Schedulable A和Schedulable B的优先级，优先级值越小，优先级越高
//  A优先级与B优先级相同，若A对应stage id越小，优先级越高

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578217885.png)


公平调度：
//  结合以上代码，我们可以比较容易看出Fair调度模式的比较逻辑：
//
//  正在运行的task个数小于   最小共享核心数的要比不小于的优先级高
//  若两者正在运行的task个数都小于最小共享核心数，则比较minShare使用率的值，
//  即runningTasks.toDouble / math.max(minShare, 1.0).toDouble，越小则优先级越高
//  若minShare使用率相同，则比较权重使用率，即runningTasks.toDouble / s.weight.toDouble，越小则优先级越高
//  如果权重使用率还相同，则比较两者的名字
//
//  对于Fair调度模式，需要先对RootPool的各个子Pool进行排序，再对子Pool中的TaskSetManagers进行排序，
//  使用的算法都是FairSchedulingAlgorithm.FairSchedulingAlgorithm


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578243915.png)

它的调度逻辑主要如下：
1. 优先看正在运行的tasks数目是否小于最小共享cores数，如果两者只有一个小于，则优先调度小于的那个，原因是既然正在运行的Tasks数目小于共享cores数，说明该节点资源比较充足，应该优先利用；
2. 如果不是只有一个的正在运行的tasks数目是否小于最小共享cores数的话，则再判断正在运行的tasks数目与最小共享cores数的比率；
3. 最后再比较权重使用率，即正在运行的tasks数目与该TaskSetManager的权重weight的比，weight代表调度池对资源获取的权重，越大需要越多的资源。

 到此为止，获得了排序好的task集合， 如果存在新加入的slave，则调用taskSet的executorAdded()方法，即TaskSetManager的executorAdded()方法，代码如下：
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578330307.png)
 
 实际方法： recomputeLocality
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578345812.png)
 
 
 def recomputeLocality() {
  //currentLocalityIndex = 0 // Index of our current locality level in validLocalityLevels
  // 它是有效位置策略级别中的索引，指示当前的位置信息。也就是我们上一个task被launched所使用的Locality Level。

  // 首先获取之前的位置Level
  // currentLocalityIndex为有效位置策略级别中的索引，默认为0
  val previousLocalityLevel = myLocalityLevels(currentLocalityIndex)

  // 计算有效的位置Level
  myLocalityLevels = computeValidLocalityLevels()

  // 获得位置策略级别的等待时间
  localityWaits = myLocalityLevels.map(getLocalityWait)

  // 设置当前使用的位置策略级别的索引
  currentLocalityIndex = getLocalityIndex(previousLocalityLevel)
}


先看这个：
val previousLocalityLevel = myLocalityLevels(currentLocalityIndex)


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578385441.png)
 
 确定在我们的任务集TaskSet中应该使用哪种位置Level，以便我们做延迟调度
computeValidLocalityLevels
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578403508.png)
 
 这里，我们先看下其中几个比较重要的数据结构。在TaskSetManager中，存在如下几个数据结构：
// 每个executor上即将被执行的tasks的映射集合
private val pendingTasksForExecutor = new HashMap[String, ArrayBuffer[Int]]

// Set of pending tasks for each host. Similar to pendingTasksForExecutor,
// but at host level.

// 每个host上即将被执行的tasks的映射集合
private val pendingTasksForHost = new HashMap[String, ArrayBuffer[Int]]

// Set of pending tasks for each rack -- similar to the above.
// 每个rack上即将被执行的tasks的映射集合
private val pendingTasksForRack = new HashMap[String, ArrayBuffer[Int]]

// Set containing pending tasks with no locality preferences.
// 存储所有没有位置信息的即将运行tasks的index索引的集合
private[scheduler] var pendingTasksWithNoPrefs = new ArrayBuffer[Int]

// Set containing all pending tasks (also used as a stack, as above).
// 存储所有即将运行tasks的index索引的集合
private val allPendingTasks = new ArrayBuffer[Int]
 
这些数据结构，存储了task与不同位置的载体的对应关系。在TaskSetManager对象被构造时，有如下代码被执行：

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578469756.png)

添加一个任务的索引到所有相关的pending-task索引列表
它是根据task的preferredLocations，来决定该往哪个数据结构存储的。
最终，将task的位置信息，存储到不同的数据结构中，方便后续任务调度的处理。

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578486308.png)
 
 我们再回来：
 
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578509795.png)
 
 
 看这句
获得位置策略级别的等待时间
localityWaits = myLocalityLevels.map(getLocalityWait)


可以通过 SparkConf 进行调整：
new SparkConf() 
.set(“spark.locality.wait”, “10”)

默认值： spark.locality.wait，默认为3s
PROCESS_LOCAL ： spark.locality.wait.process   默认为3s
NODE_LOCAL：spark.locality.wait.node
RACK_LOCAL: spark.locality.wait.rack

最后：
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578535380.png)
 
 ![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578546627.png)
 
 
 再回到方法：

org.apache.spark.scheduler.TaskSchedulerImpl#resourceOffers

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578563955.png)


org.apache.spark.scheduler.TaskSchedulerImpl#resourceOfferSingleTaskSet


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578580258.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578610247.png)


该方法的主体流程如下：
        1、标志位launchedTask初始化为false，用它来标记是否有task被成功分配或者launched；
        2、循环shuffledOffers，即每个可用executor：
             2.1、获取其executorId和host；
             2.2、如果executor上可利用cpu数目大于每个task需要的数目，则继续task分配；
             2.3、调用TaskSetManager的resourceOffer()方法，处理返回的每个TaskDescription：
                2.3.1、分配task成功，将task加入到tasks对应位置（注意，tasks为一个空的，根据shuffledOffers和其可用cores生成的有一定结构的列表）；
                2.3.2、更新taskIdToTaskSetManager、taskIdToExecutorId、executorIdToTaskCount、executorsByHost、availableCpus等数据结构；
                2.3.3、确保availableCpus(i)不小于0；
                2.3.4、标志位launchedTask设置为true；
       3、返回launchedTask。

org.apache.spark.scheduler.TaskSetManager#resourceOffer


![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578637950.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578646919.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578656415.png)

![](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1546578665753.png)

org.apache.spark.scheduler.TaskSetManager#getAllowedLocalityLevel


``` scala?linenums
/**
  * 根据当前的等待时间，根据延迟调度获取我们可以启动任务的级别。
  *
 */
private def getAllowedLocalityLevel(curTime: Long): TaskLocality.TaskLocality = {
  // Remove the scheduled or finished tasks lazily

  // 判断task是否可以被调度
  def tasksNeedToBeScheduledFrom(pendingTaskIds: ArrayBuffer[Int]): Boolean = {
    var indexOffset = pendingTaskIds.size

    // 循环
    while (indexOffset > 0) {
      // 索引递减
      indexOffset -= 1
      // 获得task索引
      val index = pendingTaskIds(indexOffset)
      // 如果对应task不存在任何运行实例，且未执行成功，可以调度，返回true
      if (copiesRunning(index) == 0 && !successful(index)) {
        return true
      } else {
        // 从pendingTaskIds中移除
        pendingTaskIds.remove(indexOffset)
      }
    }
    false
  }

  // Walk through the list of tasks that can be scheduled at each location and returns true
  // if there are any tasks that still need to be scheduled. Lazily cleans up tasks that have
  // already been scheduled.
  def moreTasksToRunIn(pendingTasks: HashMap[String, ArrayBuffer[Int]]): Boolean = {
    val emptyKeys = new ArrayBuffer[String]

    // 循环pendingTasks
    val hasTasks = pendingTasks.exists {
      case (id: String, tasks: ArrayBuffer[Int]) =>

        // 判断task是否可以被调度
        if (tasksNeedToBeScheduledFrom(tasks)) {
          true
        } else {
          emptyKeys += id
          false
        }
    }
    // The key could be executorId, host or rackId
    // 移除数据
    emptyKeys.foreach(id => pendingTasks.remove(id))
    hasTasks
  }


  // 从当前索引currentLocalityIndex开始，循环myLocalityLevels
  while (currentLocalityIndex < myLocalityLevels.length - 1) {


    // 是否存在待调度task，根据不同的Locality Level，调用moreTasksToRunIn()方法从不同的数据结构中获取，
    // NO_PREF直接看pendingTasksWithNoPrefs是否为空
    val moreTasks = myLocalityLevels(currentLocalityIndex) match {

      case TaskLocality.PROCESS_LOCAL => moreTasksToRunIn(pendingTasksForExecutor)

      case TaskLocality.NODE_LOCAL => moreTasksToRunIn(pendingTasksForHost)

      case TaskLocality.NO_PREF => pendingTasksWithNoPrefs.nonEmpty

      case TaskLocality.RACK_LOCAL => moreTasksToRunIn(pendingTasksForRack)

    }

    // 不存在可以被调度的task
    if (!moreTasks) {
      // This is a performance optimization: if there are no more tasks that can
      // be scheduled at a particular locality level, there is no point in waiting
      // for the locality wait timeout (SPARK-4939).

      // 记录lastLaunchTime
      lastLaunchTime = curTime
      logInfo(s"No tasks for locality level ${myLocalityLevels(currentLocalityIndex)}, " +
        s"so moving to locality level ${myLocalityLevels(currentLocalityIndex + 1)}")

      // 位置策略索引加1
      currentLocalityIndex += 1

    } else if (curTime - lastLaunchTime >= localityWaits(currentLocalityIndex)) {
      // Jump to the next locality level, and reset lastLaunchTime so that the next locality
      // wait timer doesn't immediately expire

      // 更新localityWaits
      lastLaunchTime += localityWaits(currentLocalityIndex)
      logInfo(s"Moving to ${myLocalityLevels(currentLocalityIndex + 1)} after waiting for " +
        s"${localityWaits(currentLocalityIndex)}ms")

      // 位置策略索引加1
      currentLocalityIndex += 1


    } else {

      // 返回当前位置策略级别
      return myLocalityLevels(currentLocalityIndex)
    }

  }
  // 返回当前位置策略级别
  myLocalityLevels(currentLocalityIndex)
}
```
在确定allowedLocality后，我们就需要调用dequeueTask()方法，出列task，进行调度。代码如下：


``` scala?linenums
/**
 * Dequeue a pending task for a given node and return its index and locality level.
 * Only search for tasks matching the given locality constraint.
 *
 * @return An option containing (task index within the task set, locality, is speculative?)
 */
private def dequeueTask(execId: String, host: String, maxLocality: TaskLocality.Value)
  : Option[(Int, TaskLocality.Value, Boolean)] =
{

  //< dequeueTaskFromList: 该方法获取list中一个可以launch的task，
  // 同时清除扫描过的已经执行的task。其实它从第二次开始首先扫描的一定是已经运行完成的task，因此是延迟清除
  // 同一个Executor，通过execId来查找相应的等待的task

  // 首先调用dequeueTaskFromList()方法，对PROCESS_LOCAL级别的task进行调度
  for (index <- dequeueTaskFromList(execId, host, getPendingTasksForExecutor(execId))) {
    return Some((index, TaskLocality.PROCESS_LOCAL, false))
  }

  // 通过主机名找到相应的Task,不过比之前的多了一步判断
  // PROCESS_LOCAL未调度到task的话，再调度NODE_LOCAL级别
  if (TaskLocality.isAllowed(maxLocality, TaskLocality.NODE_LOCAL)) {
    for (index <- dequeueTaskFromList(execId, host, getPendingTasksForHost(host))) {
      return Some((index, TaskLocality.NODE_LOCAL, false))
    }
  }

  // NODE_LOCAL未调度到task的话，再调度NO_PREF级别
  if (TaskLocality.isAllowed(maxLocality, TaskLocality.NO_PREF)) {
    // Look for noPref tasks after NODE_LOCAL for minimize cross-rack traffic
    for (index <- dequeueTaskFromList(execId, host, pendingTasksWithNoPrefs)) {
      return Some((index, TaskLocality.PROCESS_LOCAL, false))
    }
  }

  // NO_PREF未调度到task的话，再调度RACK_LOCAL级别
  if (TaskLocality.isAllowed(maxLocality, TaskLocality.RACK_LOCAL)) {
    for {
      rack <- sched.getRackForHost(host)
      index <- dequeueTaskFromList(execId, host, getPendingTasksForRack(rack))
    } {
      return Some((index, TaskLocality.RACK_LOCAL, false))
    }
  }

  // 最好是ANY级别的调度
  if (TaskLocality.isAllowed(maxLocality, TaskLocality.ANY)) {
    for (index <- dequeueTaskFromList(execId, host, allPendingTasks)) {
      return Some((index, TaskLocality.ANY, false))
    }
  }


  // find a speculative task if all others tasks have been scheduled
  // 最后没办法了，拖的时间太长了，只能启动推测执行了
  // 如果所有的class都被调度的话，寻找一个speculative task，同MapReduce的推测执行原理的思想
  dequeueSpeculativeTask(execId, host, maxLocality).map {
    case (taskIndex, allowedLocality) => (taskIndex, allowedLocality, true)}
}
```

按照PROCESS_LOCAL、NODE_LOCAL、NO_PREF、RACK_LOCAL、ANY的顺序进行调度。
得到TaskDescription



## Task启动