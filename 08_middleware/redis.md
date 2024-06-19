## redis 部署方式

* 主从模式
* 哨兵模式
* 集群模式

## 数据类型
* String， 普通key value 类型，最大512M。 分布式锁 setnx 实现。
* Hash  多个KV的集合，一般用作对象。 hset, hget, hmset, hmget, hdel 等命令操作。
* List 字符串列表，支持lpush(在投上添加),rpush(在队尾添加),lrange(获取指定范围的字串)，lrem(删除指定字串)，lset(修改指定索引的字串)，lindex(获取指定索引的字串)， lpop(弹出左侧字串)，rpop(弹出右侧字串)。场景：消息队列。
* Set 字符串的无序集合。sadd, srem, sismember, smembers, srandmember 等命令操作。场景：共同好友，黑名单。
* Zset y分值有序集合。 zadd, zrem, zrange, zrevrange, zscore, zrank, zrevrank, zcard, zcount, zincrby 等命令操作。场景：排行榜，商品推荐。

## 特殊数据类型
* HyperLogLog 基数估计，用于去重。一个可重复集合内不重复元素的个数就是基数。页面访问统计。2^64不同元素的技术，只需要费12KB内存。
* Geo 地理位置，定位、查看附近的人、朋友的定位、打车距离计算等等。 geoadd，geopos
* bitmap 位图数据结构，只有0、1 两种状态，用于统计活跃用户。