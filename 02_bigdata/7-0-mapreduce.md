## MapReduce  是什么
MapReduce 是一个经典的分布式批处理计算引擎 . 

## MapReduce 运行流程
MapReduce任务过程是分为两个处理阶段：
Map阶段：Map 阶段的主要作用是“分”，即把复杂的任务分解为若干个“简单的任务”来并行处理。Map阶段的这些任务可以并行计算，彼此间没有依赖关系。
Reduce阶段：Reduce 阶段的主要作用是 “合”，即对 Map 阶段的结果进行全局汇总 

### MapTask 执行阶段
（1）MapReduce框架使用InputFormat模块做Map前的预处理，比如验证输入的格式是否符合输入定义；然后，将输入文件切分为逻辑上的多个InputSplit。InputSplit是MapReduce对文件进行处理和运算的输入单位，只是一个逻辑概念，每个InputSplit并没有对文件进行实际切分，只是记录了要处理的数据的位置和长度，通常是 hdfs 的 一个 block。有多少个split就对应启动多少个MapTask。
（2）因为 InputSplit 是逻辑切分而非物理切分，所以还需要通过 RecordReader（RR）根据InputSplit中的信息来处理InputSplit中的具体记录，加载数据并将其转换为适合Map任务读取的键值对，输入给Map任务。
（3）Map任务会根据用户自定义的映射规则，输出一系列的<key,value>作为中间结果。
（4）为了让Reduce可以并行处理Map的结果，需要对Map的输出进行一定的分区（Partition）、排序（Sort）、合并（Combine）、归并（Merge）等操作，得到<key,value-list>形式的中间结果，再交给对应的Reduce来处理，这个过程称为Shuffle。从无序的<key,value>到有序的<key,value-list>，这个过程用Shuffle来称呼是非常形象的。
（5）Reduce 以一系列<key,value-list>中间结果作为输入，执行用户定义的逻辑，输出结果交给OutputFormat模块。
（6）OutputFormat模块会验证输出目录是否已经存在，以及输出结果类型是否符合配置文件中的配置类型，如果都满足，就输出Reduce的结果到分布式文件系统

![](https://github.com/chenxh/interviews/raw/main/imgs/mp-flow.png "")


### Shuffle 过程
Shuffle是指对Map任务输出结果进行分区、排序、合并、归并等处理并交给Reduce的过程。

![](https://github.com/chenxh/interviews/raw/main/imgs/mp-shuffle.png "")


**Map端的Shuffle过程**
* 执行 Map 任务 ，任务结果映射成多个<key, value>输出。
* 写入缓存： 每个 Map 任务分配一个缓存， 收集<key, value>结果，在写入缓存之前，key与value都会被序列化成字节数组。
* 溢写（分区、排序和合并）。
    * 分区：缓存中的数据超过阈值就会溢写到磁盘上。在溢写到磁盘之前，缓存中的数据首先会被分区，分区通过Partitioner 完成，默认 Partitioner 是 HashPartitioner， 通过对 key 进行 hash ，然后对Reduce任务的数量进行取模来分区：hash(key) mod R。
    * 排序：每个分区内的所有键值对，后台线程会根据key对它们进行内存排序，排序是MapReduce的默认操作。
    * 合并：是指将那些具有相同key的<key,value>的value加起来。如果用户定义了Combiner，才会进行合并操作。
* 文件归并：每次溢写操作都会在磁盘中生成一个新的溢写文件，在Map任务全部结束之前，系统会对所有溢写文件中的数据进行归并，生成一个大的溢写文件，这个大的溢写文件中的所有键值对也是经过分区和排序的。“归并”是指具有相同key的键值对会被归并成一个新的键值对。具体而言，若干个具有相同key的键值对<k1,v1>,<k1,v2>,…,<k1,vn>会被归并成一个新的键值对<k1,<v1,v2,…,vn>>。 如果磁盘中已经生成的溢写文件的数量超过参数min.num.spills.for.combine（默认3）的值时，会再次运行Combiner。
**Reduce端的Shuffle过程**
* 拉取数据：JobTracker监测到一个Map任务完成后，就会通知相关的Reduce任务来“领取”数据；一旦一个Reduce任务收到JobTracker的通知，它就会到该Map任务所在机器上把属于自己处理的分区数据领取到本地磁盘中。
* 归并数据
    * 从Map端“领取”的数据会被存放在Reduce任务所在机器的缓存中。
    * 缓存被占满，也会产生溢写操作。 溢写过程会归并键值。如果定义了Combiner，归并后的数据，可进行合并操作。
    * 如果缓存足够，就直接使用缓存数据给 Reduce。
* 把数据输入给Reduce任务。

 

    

## MapReduce 解决了什么问题
在各种行业中，当需要处理数据量到一定程度时，分布式处理是必须要走的路。
MapReduce 提供了一套容易编写出分布式处理程序的软件。
MapReduce模型是对大量分布式处理问题的总结和抽象，它的核心思想是分而治之，即将一个分布式计算过程拆解成两个阶段：
1. Map阶段，由多个可并行执行的Map Task构成，主要功能是，将待处理数据集按照数据量大小切分成等大的数据分片，每个分片交由一个任务处理。
2. Reduce阶段，由多个可并行执行的Reduce Task构成，主要功能是，对前一阶段中各任务产生的结果进行规约，得到最终结果。

MapReduce的出现，使得用户可以把主要精力放在设计数据处理算法上，至于其他的分布式问题，包括节点间的通信、节点失效、数据切分、任务并行化等，全部由MapReduce运行时环境完成，用户无需关心这些细节。

## MapReduce 内部原理
MapReduce 运行 Hadoop Yarn 上。
当用户向YARN中提交一个MapReduce应用程序后，YARN将分两个阶段运行该应用程序：
第一个阶段是由ResourceManager启动MRAppMaster；
第二个阶段是由MRAppMaster创建应用程序，为它申请资源，并监控它的整个运行过程，直到运行成功。

工作流程:
1. 用户向YARN集群提交应用程序，该应用程序包括以下配置信息：MRAppMaster所在jar包、启动MRAppMaster的命令及其资源需求（CPU、内存等）、用户程序jar包等。
2. ResourceManager为该应用程序分配第一个Container，并与对应的NodeManager通信，要求它在这个Container中启动应用程序的MRAppMaster。
3. MRAppMaster启动后，首先向ResourceManager注册（告之所在节点、端口号以及访问链接等），这样，用户可以直接通过ResourceManager查看应用程序的运行状态，之后，为内部Map Task和Reduce Task申请资源并运行它们，期间监控它们的运行状态，直到所有任务运行结束，即重复步骤4～7。
4. MRAppMaster采用轮询的方式通过RPC协议向ResourceManager申请和领取资源。
5. 一旦MRAppMaster申请到（部分）资源后，则通过一定的调度算法将资源分配给内部的任务，之后与对应的NodeManager通信，要求它启动这些任务。
6. NodeManager为任务准备运行环境（包括环境变量、jar包、二进制程序等），并将任务执行命令写到一个shell脚本中，并通过运行该脚本启动任务
7.  启动的Map Task或Reduce Task通过RPC协议向MRAppMaster汇报自己的状态和进度，以让MRAppMaster随时掌握各个任务的运行状态，从而可以在任务失败时触发相应的容错机制 
8. 应用程序运行完成后，MRAppMaster通过RPC向ResourceManager注销，并关闭自己.

## MapReduce 设计剖析
### 高可用设计
MapReduce 运行的状态图：

![](https://github.com/chenxh/interviews/raw/main/imgs/MapReduce-run.png "")

高可用方案如下：
1. Yarn 自身具有高可用功能，所以Resource Manager  有高可用
2. MRAppMaster由ResourceManager管理，一旦MRAppMaster因故障挂掉，ResourceManager会重新为它分配资源，并启动之。重启后的MRAppMaster需借助上次运行时记录的信息恢复状态，包括未运行、正在运行和已运行完成的任务。
3. MapTask/ReduceTask 任务由MRAppMaster管理，一旦MapTask/ReduceTask因故障挂掉或因程序bug阻塞住，MRAppMaster会为之重新申请资源并启动之。

### 性能设计
1. 数据分片： MapReduce 天然具有数据分片功能。 数据从输入到 Map Task， 再到Reduce Task 每个环境都使用 key 来对应每条数据。 MapReduce  使用分片组件 Partitioner 来分片，默认实现是HashPartitioner。
2. 数据本地性： MapReduce根据输入数据与实际分配的计算资源之间的距离将任务分成三类：node-local、rack-local和off-switch，分别表示输入数据与计算资源同节点、同机架和跨机架，当输入数据与计算资源位于不同节点上时，MapReduce需将输入数据远程拷贝到计算资源所在的节点进行处理，两者距离越远，需要的网络开销越大，因此调度器进行任务分配时尽量选择离输入数据近的节点资源。
3. 推测执行
 在分布式集群环境下，因软件Bug、负载不均衡或者资源分布不均等原因，造成同一个作业的多个任务之间运行速度不一致，有的任务运行速度明显慢于其他任务（比如某个时刻，一个作业的某个任务进度只有10%，而其他所有Task已经运行完毕），则这些任务将拖慢作业的整体执行进度。为了避免这种情况发生，运用推测执行（Speculative Execution）机制，Hadoop会为该任务启动一个备份任务，让该备份任务与原始任务同时处理一份数据，谁先运行完成，则将谁的结果作为最终结果。
    推测执行算法的核心思想是：某一时刻，判断一个任务是否拖后腿或者是否是值得为其启动备份任务，采用的方法为，先假设为其启动一个备份任务，则可估算出备份任务的完成时间estimatedEndTime2；同样地，如果按照此刻该任务的计算速度，可估算出该任务最有可能的完成时间estimatedEndTime1，这样estimatedEndTime1与estimatedEndTime2之差越大，表明为该任务启动备份任务的价值越大，则倾向于为这样的任务启动备份任务。
参数控制:
```
mapreduce.map.speculative
mapreduce.reduce.speculative
```






