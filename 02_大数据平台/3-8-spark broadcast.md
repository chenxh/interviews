## broadcast
broadcast 就是将数据从一个节点发送到其他各个节点上去。
使用场景：
1. mapOutputTracker 
2. sparkContext.broadcast() 变量。
3. 广播 rdd 信息 ，任务运行信息到 executor， 在 runTask 时使用


## broadcast 实现方式

代码中调用 broadcast 时， 首先调用 TorrentBroadcastFactory.newBroadcast 方法， 此方法直接新建 TorrentBroadcast 实例。新建TorrentBroadcast， 广播变量会先通过 writeBlocks 把数据写到 driver 的blockmanager 中。 在Executor 反序列化 Task 时， 会读取广播变量的值 broadcastVal.value 。


## TorrentBroadcast 的流程



