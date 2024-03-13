## 存储体系架构
![](https://github.com/chenxh/interviews/blob/main/imgs/spark-blockmanager.png  "spark-blockmanager")


https://zhuanlan.zhihu.com/p/338437439
## BlockManager
Spark存储体系是各个 Driver 和 Executor 实例中的BlockManager所组成的。
* driver 端的 BlockManager 持有 BlockManagerMasterEndpoint 对各个节点上的BlockManager、BlockManager与Executor的映射关系及Block位置信息（即Block所在的BlockManager）等进行管理。
* executor/driver 端的 BlockManager 持有 BlockManagerSlaveEndpoint ， BlockManagerMasterEndpoint 会向 BlockManagerSlaveEndpoint 下发删除Block、获取Block状态、获取匹配的BlockId等命令。

* MapOutputTracker:map任务输出跟踪器。
* ShuffleManager:Shuffle管理器。
* BlockTransferService：块传输服务。要用于不同阶段的任务之间的Block数据的传输与读写。例如，map任务所在节点的BlockTransferService给Shuffle对应的reduce任务提供下载map中间输出结果的服务。
* shuffleClient:Shuffle的客户端。与BlockTransferService配合使用. 可以通过Executor上的BlockTransferService提供的服务上传和下载Block。不同Executor节点上的BlockTransfer-Service和shuffleClient之间也可以互相上传、下载Block
* 