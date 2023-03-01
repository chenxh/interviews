## broadcast
broadcast 就是将数据从一个节点发送到其他各个节点上去。
使用场景：
1. mapOutputTracker 
2. sparkContext.broadcast() 变量。
3. 广播 rdd 信息 ，任务运行信息到 executor， 在 runTask 时使用


## broadcast 实现方式

代码中调用 broadcast 时， 首先调用 TorrentBroadcastFactory.newBroadcast 方法， 此方法直接新建 TorrentBroadcast 实例。新建TorrentBroadcast， 广播变量会先通过 writeBlocks 把数据写到 driver 的blockmanager 中。 在Executor 反序列化 Task 时， 会读取广播变量的值 broadcastVal.value 。


## TorrentBroadcast 的写流程 
1. 新建TorrentBroadcast 实例的时候，就会调用 writeBlocks() 方法把数据写道 BlockManager 中
2. 调用 BlockManager.putSingle 方法，把广播对象的value写到本地存储体系, 只有一个 block 。 
3. 调用TorrentBroadcast的blockifyObject方法，将对象转换成一系列的块。每个块的大小由blockSize决定，使用当前SparkEnv中的JavaSerializer组件进行序列化，使用TorrentBroadcast自身的compressionCodec进行压缩。
4. 对每个块进行如下处理：如果需要给分片广播块生成校验和，则给分片广播块生成校验和。给当前分片广播块生成分片的BroadcastBlockId，分片通过BroadcastBlockId的field属性区别，例如，piece0、piece1……调用BlockManager的putBytes方法将分片广播块以序列化方式写入Driver本地的存储体系。由于MEMORY_AND_DISK_SER对应的StorageLevel的_replication属性也固定为1，因此此处只会将分片广播块写入Driver或Executor本地的存储体系 .

**为什么要写两边？第2步putSingle 和 3，4步的分片存储**
第一步写是为了在local模式下能正常运行。

## TorrentBroadcast 的读取流程 
TorrentBroadcast.scala#readBroadcastBlock()
1. 调用BlockManager的getLocalValues方法从本地的存储系统中获取广播对象，即通过BlockManager的putSingle方法写入存储体系的广播对象，BlockManager的getLocalValues方法
2. 如果从本地的存储体系中可以获取广播对象，则调用releaseLock方法（这个锁保证当块被一个运行中的任务使用时，不能被其他任务再次使用，但是当任务运行完成时，则应该释放这个锁），释放当前块的锁并返回此广播对象。
3. 如果从本地的存储体系中没有获取到广播对象，那么说明数据是通过BlockManager的putBytes方法以序列化方式写入存储体系的。此时首先调用readBlocks方法从Driver或Executor的存储体系中获取广播块，然后调用TorrentBroadcast的unBlockifyObject方法，将一系列的分片广播块转换回原来的广播对象，最后再次调用BlockManager的putSingle方法将广播对象写入本地的存储体系，以便于当前Executor的其他任务不用再次获取广播对象。
   1. readBlocks
   2. 新建用于存储每个分片广播块的数组blocks，并获取当前SparkEnv的BlockManager组件
   3. 对各个广播分片进行随机洗牌，避免对广播块的获取出现“热点”，提升性能。对洗牌后的各个广播分片依次执行下面操作。
   4. 调用BlockManager的getLocalBytes方法从本地的存储体系中获取序列化的分片广播块，如果本地可以获取到，则将分片广播块放入blocks，并且调用releaseLock方法释放此分片广播块的锁。
   5. 如果本地没有，则调用BlockManager的getRemoteBytes方法从远端的存储体系中获取分片广播块。对于获取的分片广播块再次调用calcChecksum方法计算校验和，并将此校验和与调用writeBlocks方法时存入checksums数组的校验和进行比较。如果校验和不相同，说明块的数据有损坏，此时抛出异常。
   6. 如果校验和相同，则***调用BlockManager的putBytes方法将分片广播块写入本地存储体系，以便于当前Executor的其他任务不用再次获取分片广播块, 并且告知 blockManagerMaster***。最后将分片广播块放入blocks。

TorrentBroadcast 和 Torrent下载机制类似， 刚开始只在一个 driver/executor 存储， 另外的 driver/executor 在获取值的时候，本地有就用本地，本地没有去远程获取，获取到之后会在本地存储，存储后通知blockmaster，然后再有其它 driver/executor 使用也可以从此处获取。