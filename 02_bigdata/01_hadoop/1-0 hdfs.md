# NameNode 的架构原理
NameNode 的核心功能：管理整个 HDFS 集群的元数据，比如说文件目录树、权限的设置、副本数的设置，等等。
主要功能：1.整个文件系统的文件目录树，2.文件/目录的元信息和每个文件对应的数据块列表。3.接收用户的操作请求。
NameNode 将文件目录树 维护在内存中，每次有修改，先修改内存中的数据，然后写一条edits log。 日志存在磁盘文件中，不修改内容，只做append，效率高很多。

## edits log 无限制扩大的问题
edits log 除了写入磁盘文件中，还会写入JournalNodes 集群。Standby NameNode (从节点) 会从JournalNodes 中拉取edits log，定时生成fsimage文件。
fsimage 是一份完整的元数据，不是日志。这个就是checkpoint操作。
然后Standby NameNode把这个 fsimage 上传到 Active NameNode，接着清空掉 Active NameNode 的旧的 edits log 文件。
Active NameNode  重启时，只需要读取fsimage 文件，然后回放edits log中的操作记录即可，减少了重启时间。

## hdfs 高可用
一个是主节点对外提供服务接收请求。
从节点纯就是接收和同步主节点的 edits log 以及执行定期 checkpoint 的备节点。
他们的在内存里的元数据一模一样，当主节点挂掉，可以立即切换成从节点。
hdfs-ha.jpg


## 写edits log 的高并发解决方案
NameNode在写edits log 时，一定要保证log的顺序，需要生成txid。
如果简单的使用同步保证顺序，性能存在瓶颈。
解决方案：
double-buffer 双缓冲机制。将一块内存缓冲分成两个部分：
其中一个部分可以写入。
另外一个部分用于读取后写入磁盘和 JournalNodes。

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-doublebuffer.jpg "")


①分段加锁机制 + 内存双缓冲机制
首先各个线程依次第一次获取锁，生成顺序递增的 txid，然后将 edits log 写入内存双缓冲的区域 1，接着就立马第一次释放锁了。
后面的线程就可以再次立马第一次获取锁，然后立即写自己的 edits log 到内存缓冲。
各个线程竞争第二次获取锁，有线程获取到锁之后，就看看，有没有谁在写磁盘和网络？
如果没有，这个线程交换双缓冲的区域 1 和区域 2，接着第二次释放锁。这个过程相当快速，内存里判断几个条件，耗时不了几微秒。
好，到这一步为止，内存缓冲已经被交换了，后面的线程可以立马快速的依次获取锁，然后将 edits log 写入内存缓冲的区域 2，区域 1 中的数据被锁定了，不能写。
②多线程并发吞吐量的百倍优化
接着，之前那个幸运儿线程，将内存缓冲的区域 1 中的数据读取出来（此时没人写区域 1 了，都在写区域 2），将里面的 edtis log 都写入磁盘文件，以及通过网络写入 JournalNodes 集群。
这个过程可是很耗时的！但是做过优化了，在写磁盘和网络的过程中，是不持有锁的！
因此后面的线程可以噼里啪啦的快速的第一次获取锁后，立马写入内存缓冲的区域 2，然后释放锁。

③缓冲数据批量刷磁盘 + 网络的优化
那么在幸运儿线程吭哧吭哧把数据写磁盘和网络的过程中，排在后面的大量线程，快速的第一次获取锁，写内存缓冲区域 2，释放锁，之后，这些线程第二次获取到锁后会干嘛？
他们会发现有人在写磁盘啊，兄弟们！所以会立即休眠 1 秒，释放锁。
此时大量的线程并发过来的话，都会在这里快速的第二次获取锁，然后发现有人在写磁盘和网络，快速的释放锁，休眠。

# NameNode的HA机制

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-ha2.jpg "")



在这个架构体系涉及到Zookeeper，NameNode Active和NameNode Standby和共享存储空间，ZKFC。在整个高可用架构中，Active NameNode和Standby NameNode两台NameNode形成互备，一台处于Active状态，作为主节点，另外一台是Standby状态，作为备节点，只有主节点才能对外提供读写服务。

ZKFC（ZKFailoverContoller）作为独立的进程运行，对NameNode的主备切换进行总体控制。ZKFC能够及时加测到NameNode的健康状况，在active的NameNode故障的时候，借助Zookeeper实现自动主备选举和切换。当然，NameNode目前也支持不依赖Zookeeper的手动主备切换。Zookeeper集群主要是为控制器提供主被选举支持。

共享存储系统是NameNode实现高可用的关键部分，共享存储系统保存了NameNode运行过程中的所有产生的HDFS的元数据。active NameNode和standby NameNode通过共享存储系统实现元数据同步。在主备切换的时候，新的active NameNode在确认元数据同步之后才能继续对外提供服务。

除了通过共享存储系统共享HDFS的元数据信息之外，active NameNode和 standby NameNode还需要共享HDFS的数据块和DataNode之间的映射关系，DataNode会同时向active NameNode和standby NameNode上报数据块位置信息。

## 基于QJM的共享存储系统
共享存储系统主要是由多个JournalNode进程组成，JournalNode由奇数个组成。当Active NameNode中有事物提交，active NameNode会将editLog发给jouranlNode集群，journalNode集群通过paxos协议保证数据一致性（即：超过一半以上的jounalNode节点确认），这个数据完成了提交到共享存储。standby NameNode定期从journalNode读取editLog，合并到自己的fsimage上。总体的架构如下：

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-qjm.png "")


处于 Standby 状态的 NameNode 转换为 Active 状态的时候，有可能上一个 Active NameNode 发生了异常退出，那么 JournalNode 集群中各个 JournalNode 上的 EditLog 就可能会处于不一致的状态，所以首先要做的事情就是让 JournalNode 集群中各个节点上的 EditLog 恢复为一致。

另外如前所述，当前处于 Standby 状态的 NameNode 的内存中的文件系统镜像有很大的可能是落后于旧的 Active NameNode 的，所以在 JournalNode 集群中各个节点上的 EditLog 达成一致之后，接下来要做的事情就是从 JournalNode 集群上补齐落后的 EditLog。只有在这两步完成之后，当前新的 Active NameNode 才能安全地对外提供服务。

## HA的切换流程
HA机制更多细节，NameNode的切换流程分为以下几个步骤：
1. HealthMonitor 初始化完成之后会启动内部的线程来定时调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法，对 NameNode 的健康状态进行检测。
2. HealthMonitor 如果检测到 NameNode 的健康状态发生变化，会回调 ZKFailoverController 注册的相应方法进行处理。
3. 如果 ZKFailoverController 判断需要进行主备切换，会首先使用 ActiveStandbyElector 来进行自动的主备选举。
4. ActiveStandbyElector 与 Zookeeper 进行交互完成自动的主备选举。
5. ActiveStandbyElector 在主备选举完成后，会回调 ZKFailoverController 的相应方法来通知当前的 NameNode 成为主 NameNode 或备 NameNode。
6. ZKFailoverController 调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法将 NameNode 转换为 Active 状态或 Standby 状态。

# hdfs 读写流程
## 读流程

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-read.jpg "")


客户端打开文件系统，通过rpc的方式向NameNode获取文件快的存储位置信息，NameNode会将文件中的各个块的所有副本DataNode全部返回，这些DataNode会按照与客户端的位置的距离排序。如果客户端就是在DataNode上，客户端可以直接从本地读取文件，跳过网络IO，性能更高。客户端调研read方法，存储了文件的前几个块的地址的DFSInputStream，就会连接存储了第一个块的最近的DataNode。然后通过DFSInputStream就通过重复调用read方法，数据就从DataNode流动到可后端，当改DataNode的最后一个快读取完成了，DFSInputSteam会关闭与DataNode的连接，然后寻找下一个快的最佳节点。这个过程读客户端来说透明的，在客户端那边来看们就像是只读取了一个连续不断的流。

块是按顺序读的，通过 DFSInputStream 在 datanode 上打开新的连接去作为客户端读取的流。他也将会呼叫 namenode 来取得下一批所需要的块所在的 datanode 的位置(注意刚才说的只是从 namenode 获取前几个块的)。当客户端完成了读取，就在 FSDataInputStream 上调用 close() 方法结束整个流程。

在这个设计中一个重要的方面就是客户端直接从 DataNode 上检索数据，并通过 NameNode 指导来得到每一个块的最佳 DataNode。这种设计允许 HDFS 扩展大量的并发客户端，因为数据传输只是集群上的所有 DataNode 展开的。期间，NameNode 仅仅只需要服务于获取块位置的请求（块位置信息是存放在内存中，所以效率很高）。如果不这样设计，随着客户端数据量的增长，数据服务就会很快成为一个瓶颈。

## 写流程

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-write.jpg "")


1.通过Client向远程的NameNode发送RPC请求；
2.接收到请求后NameNode会首先判断对应的文件是否存在以及用户是否有对应的权限，成功则会为文件创建一个记录，否则会让客户端抛出异常；
3. 当客户端开始写入文件的时候，开发库会将文件切分成多个packets，并在内部以"data queue"的形式管理这些packets，并向Namenode申请新的blocks，获取用来存储replicas的合适的datanodes列表，列表的大小根据在Namenode中对replication的设置而定。
3.开始以pipeline（管道）的形式将packet写入所有的replicas中。开发库把packet以流的方式写入第一个 datanode，该datanode把该packet存储之后，再将其传递给在此pipeline中的下一个datanode，直到最后一个 datanode， 这种写数据的方式呈流水线的形式。
4.最后一个datanode成功存储之后会返回一个ack packet，在pipeline里传递至客户端，在客户端的开发库内部维护着 "ack queue"，成功收到datanode返回的ack packet后会从"ack queue"移除相应的packet。
5.如果传输过程中，有某个datanode出现了故障，那么当前的pipeline会被关闭，出现故障的datanode会从当前的 pipeline中移除，剩余的block会继续剩下的datanode中继续以pipeline的形式传输，同时Namenode会分配一个新的 datanode，保持replicas设定的数量。

## DFSOutputStream内部原理
打开一个DFSOutputStream流，Client会写数据到流内部的一个缓冲区中，然后数据被分解成多个Packet，每个Packet大小为64k字节，每个Packet又由一组chunk和这组chunk对应的checksum数据组成，默认chunk大小为512字节，每个checksum是对512字节数据计算的校验和数据。

当Client写入的字节流数据达到一个Packet的长度，这个Packet会被构建出来，然后会被放到队列dataQueue中，接着DataStreamer线程会不断地从dataQueue队列中取出Packet，发送到复制Pipeline中的第一个DataNode上，并将该Packet从dataQueue队列中移到ackQueue队列中。ResponseProcessor线程接收从Datanode发送过来的ack，如果是一个成功的ack，表示复制Pipeline中的所有Datanode都已经接收到这个Packet，ResponseProcessor线程将packet从队列ackQueue中删除。

在发送过程中，如果发生错误，所有未完成的Packet都会从ackQueue队列中移除掉，然后重新创建一个新的Pipeline，排除掉出错的那些DataNode节点，接着DataStreamer线程继续从dataQueue队列中发送Packet。

下面是DFSOutputStream的结构及其原理，从下面3个方面来描述内部流程：

![](https://github.com/chenxh/interviews/raw/main/imgs/hdfs-output.png "")


1. 创建Packet
Client写数据时，会将字节流数据缓存到内部的缓冲区中，当长度满足一个Chunk大小（512B）时，便会创建一个Packet对象，然后向该Packet对象中写Chunk Checksum校验和数据，以及实际数据块Chunk Data，校验和数据是基于实际数据块计算得到的。每次满足一个Chunk大小时，都会向Packet中写上述数据内容，直到达到一个Packet对象大小（64K），就会将该Packet对象放入到dataQueue队列中，等待DataStreamer线程取出并发送到DataNode节点。

2. 发送Packet
DataStreamer线程从dataQueue队列中取出Packet对象，放到ackQueue队列中，然后向DataNode节点发送这个Packet对象所对应的数据。

3. 接收ack
发送一个Packet数据包以后，会有一个用来接收ack的ResponseProcessor线程，如果收到成功的ack，则表示一个Packet发送成功。如果成功，则ResponseProcessor线程会将ackQueue队列中对应的Packet删除。

# HDFS的使用场景和缺点
## 使用场景
* hdfs的设计一次写入，多次读取，支持修改。
* 大文件，在hdfs中一个块通常是64M、128M、256M，小文件会占用更多的元数据存储，增加文件数据块的寻址时间。
* 延时高，批处理。
* 高容错，多副本。

## 缺点
* 延迟比较高，不适合低延迟高吞吐率的场景
* 不适合小文件，小文件会占用NameNode的大量元数据存储内存，并且增加寻址时间
* 支持并发写入，一个文件只能有一个写入者
* 不支持随机修改，仅支持append
