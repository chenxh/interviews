## spark 的 shuffle



我们将 map 端划分数据、持久化数据的过程称为 shuffle write，而将 reducer 读入数据、aggregate 数据的过程称为 shuffle read 。 


SortShuffleManager


##  ShuffleWriter

抽象类ShuffleWriter定义了将map任务的中间结果输出到磁盘上的功能规范，包括将数据写入磁盘和关闭ShuffleWriter。

MapStatus用于表示ShuffleMapTask返回给TaskScheduler的执行结果.返回ShuffleMapTask运行的位置，即所在节点的BlockManager的身份标识BlockManagerId 和 reduce任务需要拉取的Block的大小


3个子类:
SortShuffleWriter
UnsafeShuffleWriter
BypassMergeSortShuffleWriter

**如何决定使用哪个writer**
SortShuffleManager.registerShuffle() 
```
override def registerShuffle[K, V, C](
      shuffleId: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      //如果 partition 数量少于  spark.shuffle.sort.bypassMergeThreshold 定义的值，并且不需要 map-side aggregation 。
      //这时使用 BypassMergeSortShuffleWriter。 直接写分区文件，然后再连接他们， 避免两次序列化和反序列化。
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      // 如果可以用 序列化的模式缓存map端的输出，使用 UnsafeShuffleWriter
      new SerializedShuffleHandle[K, V](
        shuffleId, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      new BaseShuffleHandle(shuffleId, dependency)
    }
  }
``` 


Serializer支持relocation。Serializer支持relocation是指，Serializer可以对已经序列化的对象进行排序，这种排序起到的效果和先对数据排序再序列化一致。支持relocation的Serializer是KryoSerializer，Spark默认使用JavaSerializer，通过参数spark.serializer设置；





### SortShuffleWriter

提供了对Shuffle数据的排序功能。
使用ExternalSorter作为排序器，由于ExternalSorter底层使用了Partitioned AppendOnlyMap和PartitionedPairBuffer两种缓存，因此SortShuffleWriter还支持对Shuffle数据的聚合功能。

```
 sorter = if (dep.mapSideCombine) {
      new ExternalSorter[K, V, C](
        context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
    } else {
      // In this case we pass neither an aggregator nor an ordering to the sorter, because we don't
      // care whether the keys get sorted in each partition; that will be done on the reduce side
      // if the operation being run is sortByKey.
      new ExternalSorter[K, V, V](
        context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
    }
    sorter.insertAll(records)

    // Don't bother including the time to open the merged output file in the shuffle write time,
    // because it just opens a single file, so is typically too fast to measure accurately
    // (see SPARK-3570).
    val mapOutputWriter = shuffleExecutorComponents.createMapOutputWriter(
      dep.shuffleId, mapId, dep.partitioner.numPartitions)
    sorter.writePartitionedMapOutput(dep.shuffleId, mapId, mapOutputWriter)
    partitionLengths = mapOutputWriter.commitAllPartitions(sorter.getChecksums).getPartitionLengths
    mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths, mapId)
```

* 如果ShuffleDependency的mapSideCombine属性为true（即允许在map端进行合并），那么创建ExternalSorter时，将ShuffleDependency的aggregator和keyOrdering传递给ExternalSorter的aggregator和ordering属性，否则不进行传递。这也间接决定了External Sorter选择PartitionedAppendOnlyMap还是PartitionedPairBuffer .
* 调用ExternalSorter的insertAll方法将map任务的输出记录插入到缓存中, 必要时会 spill 到磁盘。
* 创建 MapOutputWriter ， 把 ExternalSorter 中的所有数据都写到 MapOutputWriter ，默认是磁盘
* 调用 commitAllPartitions 生成索引文件，并且返回各分区的长度。
* 生成 MapStatus 。

流程图：
![](https://github.com/chenxh/interviews/blob/main/imgs/SortShuffleWriter.jpg "SortShuffleWriter.jpg")


### BypassMergeSortShuffleWriter

绕开合并、排序的 Shuffle Writer 。



1）如果没有输出的记录，调用 commitAllPartitions 生成索引文件，offset 应该都是0。
2）创建 partitionWriters 和 partitionWriterSegments 数组。
3）迭代待输出的记录，使用分区计算器并通过每条记录的key，获取记录的分区ID，调用此分区ID对应的 DiskBlockObjectWriter 的 write 方法，向临时Shuffle文件的输出流中写入键值对。
4）调用每个分区对应的 DiskBlockObjectWriter 的 commitAndGet 方法，将临时 Shuffle 文件的输出流中的数据写入到磁盘，并将返回的 FileSegment 放入 partitionWriterSegments 数组中，以此分区ID为索引的位置。
5）将所有分区文件连接到一个合并文件中, 并且返回各分区长度。
6）创建 MapStatus 。


数据流程：

![](https://github.com/chenxh/interviews/blob/main/imgs/BypassMergeSortShuffleWriter.jpg "BypassMergeSortShuffleWriter.jpg")

### UnsafeShuffleWriter
UnsafeShuffleWriter是ShuffleWriter的实现类之一，底层使用ShuffleExternalSorter作为外部排序器，所以UnsafeShuffleWriter不具备SortShuffleWriter的聚合功能。UnsafeShuffle Writer将使用Tungsten的内存作为缓存，以提高写入磁盘的性能。


![](https://github.com/chenxh/interviews/blob/main/imgs/spark-shuffle-unsafe.svg "spark-shuffle-unsafe.svg")

首先将数据序列化，保存在MemoryBlock中。然后将该数据的地址和对应的分区索引，保存在ShuffleInMemorySorter内存中，利用ShuffleInMemorySorter根据分区排序。当内存不足时，会触发spill操作，生成spill文件。最后会将所有的spill文件合并在同一个文件里。

整个过程可以想象成归并排序。ShuffleExternalSorter负责分片的读取数据到内存，然后利用ShuffleInMemorySorter进行排序。排序之后会将结果存储到磁盘文件中。这样就会有很多个已排序的文件， UnsafeShuffleWriter会将所有的文件合并。





