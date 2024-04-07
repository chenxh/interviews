## Kafka的特性
* 高吞吐量、低延迟：kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒
* 可扩展性：kafka集群支持热扩展
* 持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
* 容错性：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）
* 高并发：支持数千个客户端同时读写

## Kafka一些重要设计思想
* Consumergroup：各个consumer可以组成一个组，每个消息只能被组中的一个consumer消费，如果一个消息可以被多个consumer消费的话，那么这些consumer必须在不同的组。
* 消息状态 : 在Kafka中，消息的状态被保存在consumer中，broker不会关心哪个消息被消费了被谁消费了，只记录一个offset值（指向partition中下一个要被消费的消息位置），这就意味着如果consumer处理不好的话，broker上的一个消息可能会被消费多次。
* 消息持久化：Kafka中会把消息持久化到本地文件系统中，并且保持极高的效率。
* 消息有效期：Kafka会长久保留其中的消息，以便consumer可以多次消费.
* 批量发送：Kafka支持Producer以消息集合为单位进行批量发送，以提高push效率.
* push-and-pull :Kafka中的Producer和consumer采用的是push-and-pull模式，即Producer只管向broker push消息，consumer只管从broker pull消息，两者对消息的生产和消费是异步的。
* Kafka集群中broker之间的关系：不是主从关系，各个broker在集群中地位一样，我们可以随意的增加或删除任何一个broker节点
* 负载均衡方面： Kafka提供了一个 metadata API来管理broker之间的负载（对Kafka0.8.x而言，对于0.7.x主要靠zookeeper来实现负载均衡）。
* 分区机制partition：Kafka的broker端支持消息分区，Producer可以决定把消息发到哪个分区，在一个分区中消息的顺序就是Producer发送消息的顺序，一个主题中可以有多个分区，具体分区的数量是可配置的。
* 离线数据装载：Kafka由于对可拓展的数据持久化的支持，它也非常适合向Hadoop或者数据仓库中进行数据装载。
* 插件支持：现在不少活跃的社区已经开发出不少插件来拓展Kafka的功能，如用来配合Storm、Hadoop、flume相关的插件。



## kafka 的架构

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka_orch.png "图片title")

* Producer：Producer即生产者，消息的产生者，是消息的入口。
* Broker：Broker是kafka实例，每个服务器上有一个或多个kafka的实例，我们姑且认为每个broker对应一台服务器。每个kafka集群内的broker都有一个不重复的编号，如图中的broker-0、broker-1等……
* Topic：消息的主题，可以理解为消息的分类，kafka的数据就保存在topic。在每个broker上都可以创建多个topic。
* Partition：Topic的分区，每个topic可以有多个分区，分区的作用是做负载，提高kafka的吞吐量。同一个topic在不同的分区的数据是不重复的，partition的表现形式就是一个一个的文件夹！
* Replication:每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的时候会选择一个备胎（Follower）上位，成为Leader。在kafka中默认副本的最大数量是10个，且副本的数量不能大于Broker的数量，follower和leader绝对是在不同的机器，同一机器对同一个分区也只可能存放一个副本（包括自己）。
* Message：每一条发送的消息主体。
* Consumer：消费者，即消息的消费方，是消息的出口。
* Consumer Group：我们可以将多个消费组组成一个消费者组，在kafka的设计中同一个分区的数据只能被消费者组中的某一个消费者消费。同一个消费者组的消费者可以消费同一个topic的不同分区的数据，这也是为了提高kafka的吞吐量！
* Zookeeper：kafka集群依赖zookeeper来保存集群的的元信息，来保证系统的可用性。


## 发送数据流程

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka_produce.png "图片title")
Producer在写入数据的时候永远的找leader，不会直接将数据写入follower 。
消息写入leader后，follower是主动的去leader进行同步的。
producer采用push模式将数据发布到broker，每条消息追加到分区中，顺序写入磁盘，所以保证同一分区内的数据是有序的


***partition的优点***
* 方便扩展：因为一个topic可以有多个partition，所以我们可以通过扩展机器去轻松的应对日益增长的数据量。
* 提高并发：以partition为读写单位，可以多个消费者同时消费数据，提高了消息的处理效率

***如何选择partition***
* partition在写入的时候可以指定需要写入的partition，如果有指定，则写入对应的partition。
* 如果没有指定partition，但是设置了数据的key，则会根据key的值hash出一个partition。
* 如果既没指定partition，又没有设置key，则会轮询选出一个partition。

***保证消息不丢失***
kafka 通过ACK应答机制保证消息不丢失
在生产者向队列写入数据的时候可以设置参数来确定是否确认kafka接收到数据，这个参数可设置的值为0、1、all。

* 0代表producer往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最高。
* 1代表producer往集群发送数据只要leader应答就可以发送下一条，只确保leader发送成功。
* all代表producer往集群发送数据需要所有的follower都完成从leader的同步才会发送下一条，确保leader发送成功和所有的副本都完成备份。安全性最高，但是效率最低。

***如果往不存在的topic写数据，kafka会自动创建topic，分区和副本的数量根据默认配置都是1***

## kafka 数据存储
kafka 通过顺序写的方式把数据保存到磁盘上。 顺序写效率比随机写高。

***Partition 结构***
partition 在磁盘上直观表现为一个文件夹。
每个topic都可以分为一个或多个partition。

![缓存](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka_partition.png "图片title")

每个partition的文件夹下面会有多组segment文件，每组segment文件又包含.index文件、.log文件、.timeindex文件（早期版本中没有）三个文件， log文件就实际是存储message的地方，而index和timeindex文件为索引文件，用于检索消息。

文件的命名是以该segment最小offset来命名的，如000.index存储offset为0~368795的消息，kafka就是利用分段+索引的方式来解决查找效率的问题

***Message结构***
log文件就实际是存储message的地方。
消息主要包含消息体、消息大小、offset、压缩类型……等等。
* offset：offset是一个占8byte的有序id号，它可以唯一确定每条消息在parition内的位置！
* 消息大小：消息大小占用4byte，用于描述消息的大小。
* 消息体：消息体存放的是实际的消息数据（被压缩过），占用的空间根据具体的消息而不一样

***存储策略***
无论消息是否被消费，kafka都会保存所有的消息.
旧数据的删除策略：
* 基于时间，默认配置是168小时（7天）。
* 基于大小，默认配置是1073741824。

kafka读取特定消息的时间复杂度是O(1)，所以这里删除过期的文件并不会提高kafka的性能

## 消费数据

消费数据时消费者主动的去kafka集群拉取消息，消费者在拉取消息的时候也是找leader去拉取。
多个消费者可以组成一个消费者组（consumer group），每个消费者组都有一个组id！同一个消费组者的消费者可以消费同一topic下不同分区的数据，但是不会组内多个消费者消费同一分区的数据.

![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka_consume.png "图片title")

图示是消费者组内的消费者小于partition数量的情况，所以会出现某个消费者消费多个partition数据的情况，消费的速度也就不及只处理一个partition的消费者的处理速度！如果是消费者组的消费者多于partition的数量，那会不会出现多个消费者消费同一个partition的数据呢？上面已经提到过不会出现这种情况！多出来的消费者不消费任何partition的数据。所以在实际的应用中，***建议消费者组的consumer的数量与partition的数量一致！***

***如何通过offset查找消息***
假如现在需要查找一个offset为368801的message。
![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka_find.png  "图片title")

1. 先找到offset的368801 message所在的segment文件（利用二分法查找），这里找到的就是在第二个segment文件。
2. 打开找到的segment中的.index文件（也就是368796.index文件，该文件起始偏移量为368796+1，我们要查找的offset为368801的message在该index内的偏移量为368796+5=368801，所以这里要查找的相对offset为5）。由于该文件采用的是稀疏索引的方式存储着相对offset及对应message物理偏移量的关系，所以直接找相对offset为5的索引找不到，这里同样利用二分法查找相对offset小于或者等于指定的相对offset的索引条目中最大的那个相对offset，所以找到的是相对offset为4的这个索引。
3. 根据找到的相对offset为4的索引确定message存储的物理偏移位置为256。打开数据文件，从位置为256的那个地方开始顺序扫描直到找到offset为368801的那条Message。


这套机制是建立在offset为有序的基础上，利用segment+有序offset+稀疏索引+二分查找+顺序查找等多种手段来高效的查找数据！至此，消费者就能拿到需要处理的数据进行处理了。那每个消费者又是怎么记录自己消费的位置呢？在早期的版本中，消费者将消费到的offset维护zookeeper中，consumer每间隔一段时间上报一次，这里容易导致重复消费，且性能不好！在新的版本中消费者消费到的offset已经直接维护在 kafka 集群的 __consumer_offsets 这个topic中！


## kafka 的 ISR机制
![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka-isr.jpg "图片title")


* AR (Assigned Replica): 分区中的所有副本统称为AR（某个池子在所有多元宇宙中的所有副本），AR=ISR+OSR。
* ISR (In-Sync Replica): 所有与Leader副本保持一定程度同步的副本（包括Leader副本在内）组成ISR，ISR中的副本是可靠的，它们跟随Leader副本的进度。
* OSR (Out-of-Sync Replica): 与Leader副本同步滞后过多的副本组成了OSR，这些副本可能由于网络故障或其他原因而无法与Leader保持同步。
* LEO (Log End Offset): 每个副本都有内部的LEO，代表当前队列消息的最后一条偏移量offset，LEO是该副本消息日志的末尾​，LEO的大小相当于当前日志分区中最后一条消息的offset值加1。分区ISR集合中的每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW，对消费者而言只能消费HW之前的消息。（将要流进池子的最新消息的offset，也就是说LEO所对应的消息还并未写入）。
* HW (High Watermark): 高水位代表所有ISR中的LEO最低的那个offset，它是消费者可见的最大消息offset，表示已经被所有ISR中的副本复制，HW表示数据在整个ISR中都得到了复制，是目前来说最可靠的数据。

解释上图的意思，在Kafka中消息会先被发送到Leader中，然后Follower再从Leader中拉取消息进行同步，同步期间内Follower副本消息相对于Leader会有一定程度上的滞后。前面所说的“一定程度上的之后”是可以通过参数来配置的，在正常情况下所有的Follower都应与Leader保持一定程度的同步，AR=ISR，OSR为空。

Partition副本消息复制流程：
![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka-isr1.jpg "图片title")


假设ISR集合中有3个副本，Leader、Follower-1和Follower-2，此时分区LEO和HW都为2，消息3和4从生产者出发之后会先存入Leader副本。

![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka-isr2.jpg "图片title")
消息写入Leader副本之后，由于Follower副本消费还未更新，所以HW依旧为2，而LEO为5说明下一条待写入的消息的offset为5。当Leader中3和4消息写入完毕之后，等待Follower-1和Follower-2来Leader中拉取增量数据。

![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka-isr3.jpg "图片title")
在消息同步过程中，由于网络环境和硬件因素等不同，副本同步效率也不尽相同。如上图在某一时刻follower的消息只同步到了3，而Leader和Follower-1的消息最新为4，那么当前的HW则为3。

![](https://raw.githubusercontent.com/chenxh/interviews/master/08_middleware/imgs/kafka-isr4.jpg "图片title")
当所有的副本都成功写入了消息3和消息4，整个分区的HW=4，LEO=5。

Kafka的Partition之间的复制机制既不是完全同步的复制，也不是单纯的异步复制。同步复制要求所有能够工作的副本都完成复制这条消息才算已经成功提交，这种方式极大的影响了性能。而在异步复制的方式下，Follower副本异步的从Leader副本中复制数据，数据只要被Leader写入副本就认为已经提交成功，这种情况下就存在如果Follower副本还没完全复制而落后于Leader副本而此时Leader副本突然宕机，则会造成数据丢失的情况发生。
