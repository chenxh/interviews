## 执行内存
spark 在任务执行过程中需要的内存。 spark 通过任务内存管理器（TaskMemoryManager）申请内存。

内存消费者
有对map任务的中间输出数据在JVM堆上进行缓存、聚合、溢出、持久化等处理的ExternalSorter
在操作系统内存中进行缓存、溢出、持久化处理的ShuffleExternalSorter，还
有将key/value对存储到连续的内存块中的RowBasedKeyValueBatch

## 内存管理器


MemoryLocation：用于表示内存的位置信息。
MemoryBlock：继承于 MemoryLocation， 包含内存块的页码



MemoryConsumer