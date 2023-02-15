## spark 的 部署架构
![](https://github.com/chenxh/interviews/blob/main/imgs/spark-deploy.png  "")

* 整个集群分为 Master 节点和 Worker 节点，相当于 Hadoop 的 Master 和 Slave 节点。
* Master 节点上常驻 Master 守护进程，负责管理全部的 Worker 节点。
* Worker 节点上常驻 Worker 守护进程，负责与 Master 节点通信并管理 executors。
* Driver 官方解释是 “The process running the main() function of the application and creating the SparkContext”。Application 就是用户自己写的 Spark 程序（driver program），比如 WordCount.scala。

* 每个 Worker 上存在一个或者多个 ExecutorBackend 进程。每个进程包含一个 Executor 对象，该对象持有一个线程池，每个线程可以执行一个 task。
* 每个 application 包含一个 driver 和多个 executors，每个 executor 里面运行的 tasks 都属于同一个 application。
* 在 Standalone 版本中，ExecutorBackend 被实例化成 CoarseGrainedExecutorBackend 进程。
* Worker 通过持有 ExecutorRunner 对象来控制 CoarseGrainedExecutorBackend 的启停。


## 执行过程
Job 的实际执行流程比，先建立逻辑执行图（或者叫数据依赖图），然后划分逻辑执行图生成 DAG 型的物理执行图，然后生成具体 task 执行。


### 逻辑执行图的生成逻辑
logical plan sample：
```
  MapPartitionsRDD[3] at groupByKey at GroupByTest.scala:51 (36 partitions)
    ShuffledRDD[2] at groupByKey at GroupByTest.scala:51 (36 partitions)
      FlatMappedRDD[1] at flatMap at GroupByTest.scala:38 (100 partitions)
        ParallelCollectionRDD[0] at parallelize at GroupByTest.scala:38 (100 partitions)
```

生成逻辑：
* 一般会从数据源创建 RDD。
* 每个 transformer 都会产生 RDD 。某些 transformation() 比较复杂，会包含多个子 transformation()，因而会生成多个 RDD
* x = rdda.transformation(rddb) (e.g., x = a.join(b)) 就表示 RDD x 同时依赖于 RDD a 和 RDD b
* partition 个数一般由用户指定，不指定的话一般取max(numPartitions[parent RDD 1], .., numPartitions[parent RDD n])

**RDD x 与其 parent RDDs 中 partition 之间是什么依赖关系？是依赖 parent RDD 中一个还是多个 partition？**
RDD x 中每个 partition 可以依赖于 parent RDD 中一个或者多个 partition。而且这个依赖可以是完全依赖或者部分依赖。部分依赖指的是 parent RDD 中某 partition 中一部分数据与 RDD x 中的一个 partition 相关，另一部分数据与 RDD x 中的另一个 partition 相关。下图展示了完全依赖（窄依赖）和部分依赖（宽依赖）。

![](https://github.com/chenxh/interviews/blob/main/imgs/Dependency.png "dependency")

在 Spark 中，完全依赖被称为 NarrowDependency，部分依赖被称为 ShuffleDependency。其实 ShuffleDependency 跟 MapReduce 中 shuffle 的数据依赖相同（mapper 将其 output 进行 partition，然后每个 reducer 会将所有 mapper 输出中属于自己的 partition 通过 HTTP fetch 得到）。

**典型的 transformation() 的计算过程及数据依赖图**
1) union(otherRDD)
![](https://github.com/chenxh/interviews/blob/main/imgs/union.png "")
union() 将两个 RDD 简单合并在一起，不改变 partition 里面的数据。RangeDependency 实际上也是 1:1，只是为了访问 union() 后的 RDD 中的 partition 方便，保留了原始 RDD 的 range 边界.
2) groupByKey(numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/groupByKey.png "groupByKey")
groupByKey() 只需要将 Key 相同的 records 聚合在一起，一个简单的 shuffle 过程就可以完成。ShuffledRDD 中的 compute() 只负责将属于每个 partition 的数据 fetch 过来，之后使用 mapPartitions() 操作（前面的 OneToOneDependency 展示过）进行 aggregate，生成 MapPartitionsRDD，到这里 groupByKey() 已经结束。最后为了统一返回值接口，将 value 中的 ArrayBuffer[] 数据结构抽象化成 Iterable[]。
ParallelCollectionRDD 是最基础的 RDD，直接从 local 数据结构 create 出的 RDD 属于这个类型，比如
```
val pairs = sc.parallelize(List(1, 2, 3, 4, 5), 3)
```

3) reduceByKey(func, numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/reduceByKey.png "reduceByKey")
reduceByKey() 相当于传统的 MapReduce，整个数据流也与 Hadoop 中的数据流基本一样。reduceByKey() 默认在 map 端开启 combine()，因此在 shuffle 之前先通过 mapPartitions 操作进行 combine，得到 MapPartitionsRDD，然后 shuffle 得到 ShuffledRDD，然后再进行 reduce（通过 aggregate + mapPartitions() 操作来实现）得到 MapPartitionsRDD。

4) distinct(numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/distinct.png "distinct")
distinct() 功能是 deduplicate RDD 中的所有的重复数据。由于重复数据可能分散在不同的 partition 里面，因此需要 shuffle 来进行 aggregate 后再去重。然而，shuffle 要求数据类型是 <K, V>。如果原始数据只有 Key（比如例子中 record 只有一个整数），那么需要补充成 <K, null>。这个补充过程由 map() 操作完成，生成 MappedRDD。然后调用上面的 reduceByKey() 来进行 shuffle，在 map 端进行 combine，然后 reduce 进一步去重，生成 MapPartitionsRDD。最后，将 <K, null> 还原成 K，仍然由 map() 完成，生成 MappedRDD。蓝色的部分就是调用的 reduceByKey()

5) cogroup(otherRDD, numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/cogroup.png "cogroup")


**对于两个或两个以上的 RDD 聚合，当且仅当聚合后的 RDD 中 partitioner 类别及 partition 个数与前面的 RDD 都相同，才会与前面的 RDD 构成 1:1 的关系。否则，只能是 ShuffleDependency。**

6) intersection(otherRDD)
![](https://github.com/chenxh/interviews/blob/main/imgs/intersection.png "intersection")
intersection() 功能是抽取出 RDD a 和 RDD b 中的公共数据。先使用 map() 将 RDD[T] 转变成 RDD[(T, null)]，这里的 T 只要不是 Array 等集合类型即可。接着，进行 a.cogroup(b)，蓝色部分与前面的 cogroup() 一样。之后再使用 filter() 过滤掉 [iter(groupA()), iter(groupB())] 中 groupA 或 groupB 为空的 records，得到 FilteredRDD。最后，使用 keys() 只保留 key 即可，得到 MappedRDD。


7) join(otherRDD, numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/join.png "join")
join() 将两个 RDD[(K, V)] 按照 SQL 中的 join 方式聚合在一起。与 intersection() 类似，首先进行 cogroup()，得到<K,  (Iterable[V1], Iterable[V2])>类型的 MappedValuesRDD，然后对 Iterable[V1] 和 Iterable[V2] 做笛卡尔集，并将集合 flat() 化。

这里给出了两个 example，第一个 example 的 RDD 1 和 RDD 2 使用 RangePartitioner 划分，而 CoGroupedRDD 使用 HashPartitioner，与 RDD 1/2 都不一样，因此是 ShuffleDependency。第二个 example 中， RDD 1 事先使用 HashPartitioner 对其 key 进行划分，得到三个 partition，与 CoGroupedRDD 使用的 HashPartitioner(3) 一致，因此数据依赖是 1:1。如果 RDD 2 事先也使用 HashPartitioner 对其 key 进行划分，得到三个 partition，那么 join() 就不存在 ShuffleDependency 了，这个 join() 也就变成了 hashjoin()。


8) sortByKey(ascending, numPartitions)
![](https://github.com/chenxh/interviews/blob/main/imgs/sortByKey.png "sortByKey")
sortByKey() 将 RDD[(K, V)] 中的 records 按 key 排序，ascending = true 表示升序，false 表示降序。目前 sortByKey() 的数据依赖很简单，先使用 shuffle 将 records 聚集在一起（放到对应的 partition 里面），然后将 partition 内的所有 records 按 key 排序，最后得到的 MapPartitionsRDD 中的 records 就有序了。

9) cartesian(otherRDD)
![](https://github.com/chenxh/interviews/blob/main/imgs/Cartesian.png "Cartesian")
Cartesian 对两个 RDD 做笛卡尔集，生成的 CartesianRDD 中 partition 个数 = partitionNum(RDD a) * partitionNum(RDD b)。

这里的依赖关系与前面的不太一样，CartesianRDD 中每个partition 依赖两个 parent RDD，而且其中每个 partition 完全依赖 RDD a 中一个 partition，同时又完全依赖 RDD b 中另一个 partition。这里没有红色箭头，因为所有依赖都是 NarrowDependency。


10) coalesce(numPartitions, shuffle = false)
![](https://github.com/chenxh/interviews/blob/main/imgs/Coalesce.png "Coalesce")

coalesce() 可以将 parent RDD 的 partition 个数进行调整，比如从 5 个减少到 3 个，或者从 5 个增加到 10 个。需要注意的是当 shuffle = false 的时候，是不能增加 partition 个数的（不能从 5 个变为 10 个）。

coalesce() 的核心问题是如何确立 CoalescedRDD 中 partition 和其 parent RDD 中 partition 的关系。

coalesce(shuffle = false) 时，由于不能进行 shuffle，**问题变为 parent RDD 中哪些partition 可以合并在一起。**合并因素除了要考虑 partition 中元素个数外，还要考虑 locality 及 balance 的问题。因此，Spark 设计了一个非常复杂的算法来解决该问题（算法部分我还没有深究）。注意Example: a.coalesce(3, shuffle = false)展示了 N:1 的 NarrowDependency。
coalesce(shuffle = true) 时，**由于可以进行 shuffle，问题变为如何将 RDD 中所有 records 平均划分到 N 个 partition 中。**很简单，在每个 partition 中，给每个 record 附加一个 key，key 递增，这样经过 hash(key) 后，key 可以被平均分配到不同的 partition 中，类似 Round-robin 算法。在第二个例子中，RDD a 中的每个元素，先被加上了递增的 key（如 MapPartitionsRDD 第二个 partition 中 (1, 3) 中的 1）。在每个 partition 中，第一个元素 (Key, Value) 中的 key 由 var position = (new Random(index)).nextInt(numPartitions);position = position + 1 计算得到，index 是该 partition 的索引，numPartitions 是 CoalescedRDD 中的 partition 个数。接下来元素的 key 是递增的，然后 shuffle 后的 ShuffledRDD 可以得到均分的 records，然后经过复杂算法来建立 ShuffledRDD 和 CoalescedRDD 之间的数据联系，最后过滤掉 key，得到 coalesce 后的结果 MappedRDD。

11) repartition(numPartitions)

等价于 coalesce(numPartitions, shuffle = true)

12) combineByKey

```
 def combineByKey[C](createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C,
      partitioner: Partitioner,
      mapSideCombine: Boolean = true,
      serializer: Serializer = null): RDD[(K, C)]
```
spark 使用 combineByKey 来执行 aggregate + compute() 的操作。


### 生成物理执行图
从后往前推算，遇到 ShuffleDependency 就断开，遇到 NarrowDependency 就将其加入该 stage。每个 stage 里面 task 的数目由该 stage 最后一个 RDD 中的 partition 个数决定。

**如果 stage 最后要产生 result，那么该 stage 里面的 task 都是 ResultTask，否则都是 ShuffleMapTask。**

之所以称为 ShuffleMapTask 是因为其计算结果需要 shuffle 到下一个 stage，本质上相当于 MapReduce 中的 mapper。ResultTask 相当于 MapReduce 中的 reducer（如果需要从 parent stage 那里 shuffle 数据），也相当于普通 mapper（如果该 stage 没有 parent stage）。

**不管是 1:1 还是 N:1 的 NarrowDependency，只要是 NarrowDependency chain，就可以进行 pipeline，生成的 task 个数与该 stage 最后一个 RDD 的 partition 个数相同。**

### 物理图的执行

**整个 computing chain 根据数据依赖关系自后向前建立，遇到 ShuffleDependency 后形成 stage。在每个 stage 中，每个 RDD 中的 compute() 调用 parentRDD.iter() 来将 parent RDDs 中的 records 一个个 fetch 过来**

### 生成 job

用户的 driver 程序中一旦出现 action()，就会生成一个 job，比如 foreach() 会调用sc.runJob(this, (iter: Iterator[T]) => iter.foreach(f))，向 DAGScheduler 提交 job。如果 driver 程序后面还有 action()，那么其他 action() 也会生成 job 提交。所以，driver 有多少个 action()，就会生成多少个 job。这就是 Spark 称 driver 程序为 application（可能包含多个 job）而不是 job 的原因。

每一个 job 包含 n 个 stage，最后一个 stage 产生 result。比如，第一章的 GroupByTest 例子中存在两个 job，一共产生了两组 result。在提交 job 过程中，DAGScheduler 会首先划分 stage，然后先提交无 parent stage 的 stages，并在提交过程中确定该 stage 的 task 个数及类型，并提交具体的 task。无 parent stage 的 stage 提交完后，依赖该 stage 的 stage 才能够提交。从 stage 和 task 的执行角度来讲，一个 stage 的 parent stages 执行完后，该 stage 才能执行。

| Action | finalRDD(records) => result | compute(results) |
|:---------| :-------|:-------|
| reduce(func) | (record1, record2) => result, (result, record i) => result | (result1, result 2) => result, (result, result i) => result
| collect() |Array[records] => result | Array[result] |
| count() | count(records) => result | sum(result) |
| foreach(f) | f(records) => result | Array[result] |
| take(n) | record (i<=n) => result | Array[result] |
| first() | record 1 => result | Array[result] |
| takeSample() | selected records => result | Array[result] |
| takeOrdered(n, [ordering]) | TopN(records) => result | TopN(results) |
| saveAsHadoopFile(path) | records => write(records) | null |
| countByKey() | (K, V) => Map(K, count(K)) | (Map, Map) => Map(K, count(K)) | 


### 提交 job 的实现细节

1. rdd.action() 会调用 `DAGScheduler.runJob(rdd, processPartition, resultHandler)` 来生成 job。
2. runJob() 会首先通过`rdd.getPartitions()`来得到 finalRDD 中应该存在的 partition 的个数和类型：Array[Partition]。然后根据 partition 个数 new 出来将来要持有 result 的数组 `Array[Result](partitions.size)`。
3. 最后调用 DAGScheduler 的`runJob(rdd, cleanedFunc, partitions, allowLocal, resultHandler)`来提交 job。cleanedFunc 是 processParittion 经过闭包清理后的结果，这样可以被序列化后传递给不同节点的 task。
4. DAGScheduler 的 runJob 继续调用`submitJob(rdd, func, partitions, allowLocal, resultHandler)` 来提交 job。
5. submitJob() 首先得到一个 jobId，然后再次包装 func，向 DAGSchedulerEventProcessActor 发送 JobSubmitted 信息，该 actor 收到信息后进一步调用`dagScheduler.handleJobSubmitted()`来处理提交的 job。之所以这么麻烦，是为了符合事件驱动模型。
6. handleJobSubmmitted() 首先调用 finalStage = newStage() 来划分 stage，然后submitStage(finalStage)。由于 finalStage 可能有 parent stages，实际先提交 parent stages，等到他们执行完，finalStage 需要再次提交执行。再次提交由 handleJobSubmmitted() 最后的 submitWaitingStages() 负责。

分析一下 newStage() 如何划分 stage：

1. 该方法在 new Stage() 的时候会调用 finalRDD 的 getParentStages()。
2. getParentStages() 从 finalRDD 出发，反向 visit 逻辑执行图，遇到 NarrowDependency 就将依赖的 RDD 加入到 stage，遇到 ShuffleDependency 切开 stage，并递归到 ShuffleDepedency 依赖的 stage。
3. 一个 ShuffleMapStage（不是最后形成 result 的 stage）形成后，会将该 stage 最后一个 RDD 注册到`MapOutputTrackerMaster.registerShuffle(shuffleDep.shuffleId, rdd.partitions.size)`，这一步很重要，因为 shuffle 过程需要 MapOutputTrackerMaster 来指示 ShuffleMapTask 输出数据的位置。

分析一下 submitStage(stage) 如何提交 stage 和 task：

1. 先确定该 stage 的 missingParentStages，使用`getMissingParentStages(stage)`。如果 parentStages 都可能已经执行过了，那么就为空了。
2. 如果 missingParentStages 不为空，那么先递归提交 missing 的 parent stages，并将自己加入到 waitingStages 里面，等到 parent stages 执行结束后，会触发提交 waitingStages 里面的 stage。
3. 如果 missingParentStages 为空，说明该 stage 可以立即执行，那么就调用`submitMissingTasks(stage, jobId)`来生成和提交具体的 task。如果 stage 是 ShuffleMapStage，那么 new 出来与该 stage 最后一个 RDD 的 partition 数相同的 ShuffleMapTasks。如果 stage 是 ResultStage，那么 new 出来与 stage 最后一个 RDD 的 partition 个数相同的 ResultTasks。一个 stage 里面的 task 组成一个 TaskSet，最后调用`taskScheduler.submitTasks(taskSet)`来提交一整个 taskSet。
4. 这个 taskScheduler 类型是 TaskSchedulerImpl，在 submitTasks() 里面，每一个 taskSet 被包装成 manager: TaskSetMananger，然后交给`schedulableBuilder.addTaskSetManager(manager)`。schedulableBuilder 可以是 FIFOSchedulableBuilder 或者 FairSchedulableBuilder 调度器。submitTasks() 最后一步是通知`backend.reviveOffers()`去执行 task，backend 的类型是 SchedulerBackend。如果在集群上运行，那么这个 backend 类型是 SparkDeploySchedulerBackend。
5. SparkDeploySchedulerBackend 是 CoarseGrainedSchedulerBackend 的子类，`backend.reviveOffers()`其实是向 DriverActor 发送 ReviveOffers 信息。SparkDeploySchedulerBackend 在 start() 的时候，会启动 DriverActor。DriverActor 收到 ReviveOffers 消息后，会调用`launchTasks(scheduler.resourceOffers(Seq(new WorkerOffer(executorId, executorHost(executorId), freeCores(executorId)))))` 来 launch tasks。scheduler 就是 TaskSchedulerImpl。`scheduler.resourceOffers()`从 FIFO 或者 Fair 调度器那里获得排序后的 TaskSetManager，并经过`TaskSchedulerImpl.resourceOffer()`，考虑 locality 等因素来确定 task 的全部信息 TaskDescription。调度细节这里暂不讨论。
6. DriverActor 中的 launchTasks() 将每个 task 序列化，如果序列化大小不超过 Akka 的 akkaFrameSize，那么直接将 task 送到 executor 那里执行`executorActor(task.executorId) ! LaunchTask(new SerializableBuffer(serializedTask))`。


