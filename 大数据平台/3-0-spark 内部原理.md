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

