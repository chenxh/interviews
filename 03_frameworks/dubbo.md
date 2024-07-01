### Dubbo一般使用什么注册中心？还有别的选择吗？
推荐使用 Zookeeper 作为注册中心，还有 Redis、Multicast、Simple 注册中心，但不推荐。

### Dubbo 默认使用什么序列化框架，你知道的还有哪些？
推荐使用 Hessian 序列化，还有 Duddo、FastJson、Java 自带序列化。

### Dubbo 集群容错有几种方案

Dubbo 集群容错有以下几种方案：

1. Failover 失败自动切换：当出现失败，重试其它服务器，通常用于读操作，不建议用于写操作，因为写操作通常会引起数据不一致。
2. Failfast 快速失败：只发起一次调用，失败立即报错，通常用于非幂等性的写操作。
3. Failback 失败自动恢复：后台定时检查，发现失败，重试，通常用于写操作，但需要注意的是，如果网络不通，可能会造成调用延迟。
4. Forking 并行调用：多个服务器并行调用，只要有一个成功即返回，通常用于读操作。
5. Broadcast 广播调用：所有服务器都调用一遍，如果有成功响应则返回，通常用于通知所有服务器更新缓存或日志。

### Dubbo 服务降级怎么做？
以通过 dubbo:reference 中设置 mock=“return null”。mock 的值也可以修改为 true，然后再跟接口同一个路径下实现一个 Mock 类，命名规则是 “接口名称+Mock” 后缀。然后在 Mock 类里实现自己的降级逻辑.

### Dubbo 超时设置有哪些方式？

Dubbo 超时设置有以下几种方式：

1. 服务端设置超时：在服务端的配置文件中设置超时时间，比如 dubbo.service.timeout=5000，表示服务端的超时时间为 5 秒。
2. 客户端设置超时：在客户端的配置文件中设置超时时间，比如 dubbo.reference.timeout=5000，表示客户端的超时时间为 5 秒。
3. 方法级设置超时：在服务端的接口方法上设置超时时间，比如 @Timeout(5000)，表示该方法的超时时间为 5 秒。
4. 服务端设置重试次数：在服务端的配置文件中设置重试次数，比如 dubbo.service.retries=2，表示服务端的重试次数为 2 次。
5. 客户端设置重试次数：在客户端的配置文件中设置重试次数，比如 dubbo.reference.retries=2，表示客户端的重试次数为 2 次。


### Dubbo Monitor 实现原理？
Consumer 端在发起调用之前会先走 filter 链；provider 端在接收到请求时也是先走 filter 链，然后才进行真正的业务逻辑处理。默认情况下，在 consumer 和 provider 的 filter 链中都会有 Monitorfilter。

1、MonitorFilter 向 DubboMonitor 发送数据

2、DubboMonitor 将数据进行聚合后（默认聚合 1min 中的统计数据）暂存到ConcurrentMap<Statistics, AtomicReference> statisticsMap，然后使用一个含有 3 个线程（线程名字：DubboMonitorSendTimer）的线程池每隔 1min 钟，调用 SimpleMonitorService 遍历发送 statisticsMap 中的统计数据，每发送完毕一个，就重置当前的 Statistics 的 AtomicReference

3、SimpleMonitorService 将这些聚合数据塞入 BlockingQueue queue 中（队列大写为 100000）

4、SimpleMonitorService 使用一个后台线程（线程名为：DubboMonitorAsyncWriteLogThread）将 queue 中的数据写入文件（该线程以死循环的形式来写）

5、SimpleMonitorService 还会使用一个含有 1 个线程（线程名字：DubboMonitorTimer）的线程池每隔 5min 钟，将文件中的统计数据画成图表.
