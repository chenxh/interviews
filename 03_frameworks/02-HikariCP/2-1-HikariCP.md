## HikariCP性能y优化

* HikariCP通过优化(concurrentBag，FastList )集合来提高并发的读写效率。
* HikariCP使用threadlocal缓存连接及大量使用CAS的机制，最大限度的避免lock。单可能带来cpu使用率的上升。
* 从字节码的维度优化代码(小于35个字节的函数会优化)。 (default inline threshold for a JVM running the server Hotspot compiler is 35 bytecodes ）让方法尽量在35个字节码一下，来提升jvm的处理效率


## HikariCP中的连接取用流程

HikariDataSource.getConnection()
    -> HikariPool.getConnection()
        -> ConcurrentBag.borrow() 
            如果-> HikariPool(IBagStateListener).addBagItem()
                -> addConnectionExecutor.submit()
                    -> PoolEntryCreator.call()
                        -> connectionBag.add(poolEntry);
        <- ConcurrentBag.handoffQueue.poll


 **ConcurrentBag.borrow()**
 ConcurrentBag内部同时使用了ThreadLocal和CopyOnWriteArrayList来存储元素，其中CopyOnWriteArrayList是线程共享的。
 ConcurrentBag采用了queue-stealing的机制获取元素：首先尝试从ThreadLocal中获取属于当前线程的元素来避免锁竞争，如果没有可用元素则扫描公共集合、再次从共享的CopyOnWriteArrayList中获取。
 在借用线程没有属于自己的时候,ThreadLocal列表中没有被使用的items，是可以被“窃取”的 。 
 其使用专门的 AbstractQueuedLongSynchronizer 来管理跨线程信号，这是一个"lock-less“的实现。 
 这里要特别注意的是，ConcurrentBag中通过borrow方法进行数据资源借用，通过requite方法进行资源回收，注意其中borrow方法只提供对象引用，不移除对象。
 所以从bag中“借用”的items实际上并没有从任何集合中删除，因此即使引用废弃了，垃圾收集也不会发生。因此使用时通过borrow取出的对象必须通过requite方法进行放回，否则会导致内存泄露，只有"remove"方法才能完全从bag中删除一个对象。


 borrow 的流程
 1. 首先尝试从ThreadLocal中获取属于当前线程的元素来避免锁竞争
 2. 如果没有可用元素则扫描公共集合、再次从共享的CopyOnWriteArrayList中获取
 3. 如果还没有 调用 IBagStateListener 的 addBagItem（） 添加，IBagStateListener的实现类是 HikariPool  ， 然后在 handoffQueue.poll 队列等待。  
 4. 如果 addBagItem（）创建好了， 就会向 handoffQueue 中添加对象。 ConcurrentBag.borrow 中会得到新增的连接 poolentry。








