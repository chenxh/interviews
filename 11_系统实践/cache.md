## guava



## LRU
LRU（Least Recently Used）缓存机制. 从数据集中，挑选最近最少使用的数据淘汰,最后一次使用时间最早得数据被淘汰。


accessOrder参数开启为true。
LinkedHashMap removeEldestEntry

## LFU
Least Frequently Used
数据集中，挑选最不经常使用的数据淘汰。



## ​ CacheAside旁路缓存

旁路缓存策略以数据库中的数据为准，缓存中的数据是按需加载的，它可以分为读策略和写策略。

读策略：

从缓存中读取数据；如果缓存命中，则直接返回数据；如果缓存不命中，则从数据库中查询数据；查询到数据后，将数据写入到缓存中，并且返回给用户。

写策略：

更新数据库中的记录；删除缓存记录。


## redis 部署方式

https://zhuanlan.zhihu.com/p/685272491

**主从模式**
**哨兵模式**
一个或则多个sentinel实例组成sentinel集群，监控redis主节点是否故障，当故障发生时，由sentinel集群中的另一个sentinel节点接管主节点的工作，并通知其他的sentinel节点进行转移。
sentinel 之前 leader 故障，会自动选举出新的 leader。使用 raft 协议。

**集群模式**
多个 master 和 多个 slave 组成集群，数据复制采用异步方式。 数据key 采用 hash 算法，将 key 映射到不同的master节点上, 无论请求哪个master都会转发请求到对应的master节点。


## redis 事务
multi 开启事务，exec 执行事务，discard 取消事务。
watch key 可以监视一个或多个key，事务开始时，如果监视的key被其他客户端改变，事务将被打断。

## 分布式锁








