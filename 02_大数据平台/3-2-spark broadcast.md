## broadcast
broadcast 就是将数据从一个节点发送到其他各个节点上去。
使用场景：
1. mapOutputTracker 中
2. sparkContext.broadcast() 变量。
3. 广播 rdd 信息 ，任务运行信息到 executor， 在 runTask 时使用


## broadcast 实现方式

代码中调用 broadcast 时， 首先调用 TorrentBroadcastFactory.newBroadcast 方法， 此方法直接新建 TorrentBroadcast 实例。新建实例时， 广播变量会先通过 writeBlocks 方法写道 driver 的blockmanager 中。 在反序列化 Task 时， 会读取广播变量的值（readBlocks）。

**task 如何生成， 广播变量如何在 executor 中获取？**