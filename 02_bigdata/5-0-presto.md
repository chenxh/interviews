## Presto 
Presto 是由 Facebook 开源的大数据分布式 SQL 查询引擎，适用于交互式分析查询，可支持众多的数据源，包括 HDFS，RDBMS，KAFKA 等，而且提供了非常友好的接口开发数据源连接器。

## 特点及场景介绍

* 多源、即席: 多源就是它可以支持跨不同数据源的联邦查询，即席即实时计算，将要做的查询任务实时拉取到本地进行现场计算，然后返回计算结果.
* 多租户：它支持并发执行数百个内存、I/O 以及 CPU 密集型的负载查询，并支持集群规模扩展到上千个节点；
* 联邦查询：它可以由开发者利用开放接口自定义开发针对不同数据源的连接器(Connector),从而支持跨多种不同数据源的联邦数据查询；
* 内在特性：为了保证引擎的高效性，Presto 还进行了一些优化，例如基于 JVM 运行，Code-Generation 等。

Presto之所以能在各个内存计算型数据库中脱颖而出，在于以下几点：

清晰的架构，是一个能够独立运行的系统，不依赖于任何其他外部系统。例如调度，presto自身提供了对集群的监控，可以根据监控信息完成调度。

简单的数据结构，列式存储，逻辑行，大部分数据都可以轻易的转化成presto所需要的这种数据结构。

丰富的插件接口，完美对接外部存储系统，或者添加自定义的函数。

## 整体架构

![](https://github.com/chenxh/interviews/raw/main/imgs/presto.png "")

Presto 主要是由 Client、Coordinator、Worker 以及 Connector 等几部分构成.


1.SQL 语句提交：

用户或应用通过 Presto 的 JDBC 接口或者 CLI 来提交 SQL 查询，提交的 SQL 最终传递给 Coordinator 进行下一步处理;

2.词/语法分析：

首先会对接收到的查询语句进行词法分析和语法分析，形成一棵抽象语法树。
然后，会通过分析抽象语法树来形成逻辑查询计划。

```
select orders.orderkey, sum(tax)
from orders
left join line_item
    on orders.orderkey = lineitem.orderkey
where discount=0
group by orders.orderkey
```

3.生成逻辑计划：
上面是 TPC-H 测试基准中的一条 SQL 语句，表达的是两表连接同时带有分组聚合计算的例子，经过词法语法分析后，得到 AST，然后进一步分析得到如下的逻辑计划。
![](https://github.com/chenxh/interviews/raw/main/imgs/presto-plan.png "")




4.查询优化：
Coordinator 将一系列的优化策略（例如剪枝操作、谓词下推、条件下推等）应用于与逻辑计划的各个子计划，从而将逻辑计划转换成更加适合物理执行的结构，形成更加高效的执行策略。

下面具体来说说优化器在几个方面所做的工作：

（1）自适应：Presto 的 Connector 可以通过 Data Layout API 提供数据的物理分布信息（例如数据的位置、分区、排序、分组以及索引等属性），如果一个表有多种不同的数据存储分布方式，Connector 也可以将所有的数据布局全部返回，这样 Presto 优化器就可以根据 query 的特点来选择最高效的数据分布来读取数据并进行处理。

（2）谓词下推：谓词下推是一个应用非常普遍的优化方式，就是将一些条件或者列尽可能的下推到叶子结点，最终将这些交给数据源去执行，从而可以大大减少计算引擎和数据源之间的 I/O，提高效率

![](https://github.com/chenxh/interviews/raw/main/imgs/presto-plan2.png "")

（3）节点间并行：不同 stage 之间的数据 shuffle 会带来很大的内存和 CPU 开销，因此，将 shuffle 数优化到最小是一个非常重要的目标。围绕这个目标，Presto 可以借助一下两类信息：

数据布局信息：上面我们提到的数据物理分布信息同样可以用在这里以减少 shuffle 数。例如，如果进行 join 连接的两个表的字段同属于分区字段，则可以将连接操作在在各个节点分别进行，从而可以大大减少数据的 shuffle。

再比如两个表的连接键加了索引，可以考虑采用嵌套循环的连接策略。

（4）节点内并行：优化器通过在节点内部使用多线程的方式来提高节点内对并行度，延迟更小且会比节点间并行效率更高。

交互式分析：交互式查询的负载大部分是一次执行的短查询，查询负载一般不会经过优化，这就会导致数据倾斜的现象时有发生。典型的表现为少量的节点被分到了大量的数据。

批量 ETL：这类的查询特点是任务会不加过滤的从叶子结点拉取大量的数据到上层节点进行转换操作，致使上层节点压力非常大。

针对以上两种场景遇到的问题，引擎可以通过多线程来运行单个操作符序列（或 pipeline），如图 5 所示的，pipeline1 和 2 通过多线程并行执行来加速 build 端的 hash-join。

![](https://github.com/chenxh/interviews/raw/main/imgs/presto-hashjoin.png "")

当然，除了上述列举的 Presto 优化器已经实现的优化策略，Presto 也正在积极探索 Cascades framework，相信未来优化器会得到进一步的改进。


5.容错

Presto 可以对一些临时的报错采用低级别的重试来恢复。Presto 依靠的是客户端的自动重跑失败查询。内嵌容错机制来解决 coordinator 或者 worker 节点坏掉的情况目前Presto支持的并不理想。

标准检查点或者部分修复技术是计算代价比较高的，而且很难在这种一旦结果可用就返回给客户端（即时查询类）的系统中实现。



## 资源和调度

![](https://github.com/chenxh/interviews/raw/main/imgs/presto-arch.png "")
Presto查询引擎是一个Master-Slave的架构，由一个Coordinator节点，一个Discovery Server节点，多个Worker节点组成，Discovery Server通常内嵌于Coordinator节点中。

Coordinator负责解析SQL语句，生成执行计划，分发执行任务给Worker节点执行。

Worker节点负责实际执行查询任务。***Worker节点启动后向Discovery Server服务注册，Coordinator从Discovery Server获得可以正常工作的Worker节点***。如果配置了Hive Connector，需要配置一个Hive MetaStore服务为Presto提供Hive元信息，Worker节点与HDFS交互读取数据。


### Presto的服务进程
Presto集群中有两种进程，Coordinator服务进程和worker服务进程。coordinator主要作用是接收查询请求，解析查询语句，生成查询执行计划，任务调度和worker管理。worker服务进程执行被分解的查询执行任务task。

Coordinator 服务进程部署在集群中的单独节点之中，是整个presto集群的管理节点，主要作用是接收查询请求，解析查询语句，生成查询执行计划Stage和Task并对生成的Task进行任务调度，和worker管理。Coordinator进程是整个Presto集群的master进程，需要与worker进行通信，获取最新的worker信息，有需要和client通信，接收查询请求。Coordinator提供REST服务来完成这些工作。

Presto集群中存在一个Coordinator和多个Worker节点，每个Worker节点上都会存在一个worker服务进程，主要进行数据的处理以及Task的执行。worker服务进程每隔一定的时间会发送心跳包给Coordinator。Coordinator接收到查询请求后会从当前存活的worker中选择合适的节点运行task。


![](https://github.com/chenxh/interviews/raw/main/imgs/presto-process.png "")

上图展示了从宏观层面概括了Presto的集群组件：1个coordinator，多个worker节点。用户通过客户端连接到coordinator，可以短可以是JDBC驱动或者Presto命令行cli。

Presto是一个分布式的SQL查询引擎，组装了多个并行计算的数据库和查询引擎（这就是MPP模型的定义）。Presto不是依赖单机环境的垂直扩展性。她有能力在水平方向，把所有的处理分布到集群内的各个机器上。这意味着你可以通过添加更多节点来获得更大的处理能力。

利用这种架构，Presto查询引擎能够并行的在集群的各个机器上，处理大规模数据的SQL查询。Presto在每个节点上都是单进程的服务。多个节点都运行Presto，相互之间通过配置相互协作，组成了一个完整的Presto集群。


### Coordinator
Coordinator的作用是：

* 从用户获得SQL语句

* 解析SQL语句

* 规划查询的执行计划

* 管理worker节点状态

Coordinator是Presto集群的大脑，并且是负责和客户端沟通。用户通过PrestoCLI、JDBC、ODBC驱动、其他语言工具库等工具和coordinator进行交互。Coordinator从客户端接受SQL语句，例如select语句，才能进行计算。

每个Presto集群必须有一个coordinator，可以有一个或多个worker。在开发和测试环境中，一个Presto进程可以同时配置成两种角色。

Coordinator追踪每个worker上的活动，并且协调查询的执行过程。

Coordinator给查询创建了一个包含多阶段的逻辑模型，一旦接受了SQL语句，Coordinator就负责解析、分析、规划、调度查询在多个worker节点上的执行过程，语句被翻译成一系列的任务，跑在多个worker节点上。

worker一边处理数据，结果会被coordinator拿走并且放到output缓存区上，暴露给客户端。

一旦输出缓冲区被客户完全读取，coordinator会代表客户端向worker读取更多数据。

worker节点，和数据源打交道，从数据源获取数据。因此，客户端源源不断的读取数据，数据源源源不断的提供数据，直到查询执行结束。

Coordinator通过基于HTTP的协议和worker、客户端之间进行通信。


### Workers
Presto的worker是Presto集群中的一个服务。它负责运行coordinator指派给它的任务，并处理数据。worker节点通过连接器（connector）向数据源获取数据，并且相互之间可以交换数据。最终结果会传递给coordinator。coordinator负责从worker获取最终结果，并传递给客户端。

Worker之间的通信、worker和coordinator之间的通信采用基于HTTP的协议。下图展示了多个worker如何从数据源获取数据，并且合作处理数据的流程。直到某一个worker把数据提供给了coordinator。


### 查询调度
Presto 通过 Coordinator 将 stage 以 task 的形式分发到 worker 节点，coordinator 将 task 以 stage 为单位进行串联，通过将不同 stage 按照先后执行顺序串联成一棵执行树，确保数据流能够顺着 stage 进行流动。

Presto 引擎处理一条查询需要进行两套调度，第一套是如何调度 stage 的执行顺序，第二套是判断每个 stage 有多少需要调度的 task 以及每个 task 应该分发到哪个 worker 节点上进行处理。

**（1）stage 调度**

Presto 支持两种 stage 调度策略：All-at-once 和 Phased 两种。All-at- once 策略针对所有的 stage 进行统一调度，不管 stage 之间的数据流顺序，只要该 stage 里的 task 数据准备好了就可以进行处理；Phased 策略是需要以 stage 调度的有向图为依据按序执行，只要前序任务执行完毕开会开始后续任务的调度执行。例如一个 hash-join 操作，在 hash 表没有准备好之前，Presto 不会调度 left side 表。

**（2）task 调度**

在进行 task 调度的时候，调度器会首先区分 task 所在的 stage 是哪一类 stage：Leaf Stage 和 intermediate stage。Leaf Stage 负责通过 Connector 从数据源读取数据，intermediate stage 负责处理来此其他上游 stage 的中间结果；

* leaf stages：在分发 leaf stages 中的 task 到 worker 节点的时候需要考虑网络和 connector 的限制。例如蚕蛹 shared- nothing 部署的时候，worker 节点和存储是同地协作，这时候调度器就可以根据 connector data Layout API 来决定将 task 分发到哪些 worker 节点。资料表明在一个生产集群大部分的 CPU 消耗都是花费在了对从 connector 读取到的数据的解压缩、编码、过滤以及转换等操作上，因此对于此类操作，要尽可能的提高并行度，调动所有的 worker 节点来并行处理。

* intermediate stages：这里的 task 原则上可以被分发到任意的 worker 节点，但是 Presto 引擎仍然需要考虑每个 stage 的 task 数量，这也会取决于一些相关配置，当然，有时候引擎也可以在运行的时候动态改变 task 数。

**（3）split 调度**

当 Leaf stage 中的一个 task 在一个工作节点开始执行的时候，它会收到一个或多个 split 分片，不同 connector 的 split 分片所包含的信息也不一样，最简单的比如一个分片会包含该分片 IP 以及该分片相对于整个文件的偏移量。对于 Redis 这类的键值数据库，一个分片可能包含表信息、键值格式以及要查询的主机列表。Leaf stage 中的 task 必须分配一个或多个 split 才能够运行，而 intermediate stage 中的 task 则不需要。

**（4）split 分配**
当 task 任务分配到各个工作节点后，coordinator 就开始给每个 task 分配 split 了。Presto 引擎要求 Connector 将小批量的 split 以懒加载的方式分配给 task。这是一个非常好的特点，会有如下几个方面的优点：

* 解耦时间：将前期的 split 准备工作与实际的查询执行时间分开；

* 减少不必要的数据加载：有时候一个查询可能刚出结果但是还没完全查询完就被取消了，或者会通过一些 limit 条件限制查询到部分数据就结束了，这样的懒加载方式可以很好的避免过多加载数据；

* 维护 split 队列：工作节点会为分配到工作进程的 split 维护一个队列，Coordinator 会将新的 split 分配给具有最短队列的 task，Coordinator 分给最短的。

* 减少元数据维护：这种方式可以避免在查询的时候将所有元数据都维护在内存中，例如对于 Hive Connector 来讲，处理 Hive 查询的时候可能会产生百万级的 split，这样就很容易把 Coordinator 的内存给打满。当然，这种方式也不是没有缺点，他的缺点是可能会导致难以准确估计和报告查询进度。


## 资源管理
Presto 适用于多租户部署的一个很重要的因素就是它完全整合了细粒度资源管理系统。一个单集群可以并发执行上百条查询以及最大化的利用 CPU、IO 和内存资源。

### CPU 调度
Presto 首要任务是优化所有集群的吞吐量，例如在处理数据时的 CPU 总利用量。本地（节点级别）调度又为低成本的计算任务的周转时间优化到更低，以及对于具有相似 CPU 需求的任务采取 CPU 公平调度策略。一个 task 的资源使用是这个线程下所有 split 的执行时间的累计，为了最小化协调时间，Presto 的 CPU 使用最小单位为 task 级别并且进行节点本地调度。

***Presto 通过在每个节点并发调度任务来实现多租户，并且使用合作的多任务模型。任何一个 split 任务在一个运行线程中只能占中最大 1 秒钟时长，超时之后就要放弃该线程重新回到队列。如果该任务的缓冲区满了或者 OOM 了，即使还没有到达占用时间也会被切换至另一个任务，从而最大化 CPU 资源的利用。***

当一个 split 离开了运行线程，Presto 需要去定哪一个 task（包含一个或多个 split）排在下一位运行。

Presto 通过合计每个 task 任务的总 CPU 使用时间，从而将他们分到五个不同等级的队列而不是仅仅通过提前预测一个新的查询所需的时间的方式。如果累积的 Cpu 使用时间越多，那么它的分层会越高。Presto 会为每一个曾分配一定的 CPU 总占用时间。

调度器也会自适应的处理一些情况，如果一个操作占用超时，调度器会记录他实际占用线程的时长，并且会临时减少它接下来的执行次数。这种方式有利于处理多种多样的查询类型。给一些低耗时的任务更高的优先级，这也符合低耗时任务往往期望尽快处理完成，而高耗时的任务对时间敏感性低的实际。


### 内存管理

**1.内存池**

在 Presto 中，内存被分成用户内存和系统内存，这两种内存被保存在内存池中。用户内存是指用户可以仅根据系统的基本知识或输入数据进行推理的内存使用情况(例如，聚合的内存使用与其基数成比例)。另一方面，系统内存是实现决策(例如 shuffle 缓冲区)的副产品，可能与查询和输入数据量无关。换句话说，用户内存是与任务运行有关的，我们可以通过自己的程序推算出来运行时会用到的内存，系统内存可能更多的是一些不可变的。

Presto 引擎对单独对用户内存和总的内存（用户+系统）进行不同的规则限制，如果一个查询超过了全局总内存或者单个节点内存限制，这个查询将会被杀掉。当一个节点的内存耗尽时，该查询的预留内存会因为任务停止而被阻塞。

有时候，集群的内存可能会因为数据倾斜等原因造成内存不能充分利用，那么 Presto 提供了两种机制来缓解这种问题--溢写和保留池。

**2.溢写**

当某一个节点内存用完的时候，引擎会启动内存回收程序，现将执行的任务序列进行升序排序，然后找到合适的 task 任务进行内存回收（也就是将状态进行溢写磁盘），直到有足够的内存来提供给任务序列的后一个请求。

**3.预留池**

如果集群的没有配置溢写策略，那么当一个节点内存用完或者没有可回收的内存的时候，预留内存机制就来解除集群阻塞了。这种策略下，查询内存池被进一步分成了两个池：普通池和预留池。这样当一个查询把普通池的内存资源用完之后，会得到所有节点的预留池内存资源的继续加持，这样这个查询的内存资源使用量就是普通池资源和预留池资源的加和。为了避免死锁，一个集群中同一时间只有一个查询可以使用预留池资源，其他的任务的预留池资源申请会被阻塞。这在某种情况下是优点浪费，集群可以考虑配置一下去杀死这个查询而不是阻塞大部分节点。


## Presto调优
**合理设置分区**
与Hive类似，Presto会根据元信息读取分区数据，合理的分区能减少Presto数据读取量，提升查询性能。

**使用列式存储**
Presto对ORC文件读取做了特定优化，因此在Hive中创建Presto使用的表时，建议采用ORC格式存储。相对于Parquet，Presto对ORC支持更好。

**使用压缩**
数据压缩可以减少节点间数据传输对IO带宽压力，对于即席查询需要快速解压，建议采用snappy压缩

**预排序**
对于已经排序的数据，在查询的数据过滤阶段，ORC格式支持跳过读取不必要的数据。比如对于经常需要过滤的字段可以预先排序。


**内存调优**
Presto有三种内存池，分别为GENERAL_POOL、RESERVED_POOL、SYSTEM_POOL。

GENERAL_POOL：用于普通查询的 physical operators。GENERAL_POOL 值为 总内存（Xmx 值）- 预留的（max-memory-per-node）- 系统的（0.4 * Xmx）。

SYSTEM_POOL：系统预留内存，用于读写 buffer，worker 初始化以及执行任务必要的内存。大小由 config.properties 里的 resources.reserved-system-memory 指定。默认值为 JVM max memory * 0.4。

RESERVED_POOL：大部分时间里是不参与计算的，只有当同时满足如下情形下，才会被使用，然后从所有查询里获取占用内存最大的那个查询，然后将该查询放到 RESERVED_POOL 里执行，同时注意 RESERVED_POOL 只能用于一个 Query。大小由 config.properties 里的 query.max-memory-per-node 指定，默认值为：JVM max memory * 0.1。

这三个内存池占用的内存大小是由下面算法进行分配的：

```
builder.put(RESERVED_POOL, new MemoryPool(RESERVED_POOL, config.getMaxQueryMemoryPerNode()));
builder.put(SYSTEM_POOL, new MemoryPool(SYSTEM_POOL, systemMemoryConfig.getReservedSystemMemory()));
long maxHeap = Runtime.getRuntime().maxMemory();
maxMemory = new DataSize(maxHeap - systemMemoryConfig.getReservedSystemMemory().toBytes(), BYTE);
DataSize generalPoolSize = new DataSize(Math.max(0, maxMemory.toBytes() - config.getMaxQueryMemoryPerNode().toBytes()), BYTE);
builder.put(GENERAL_POOL, new MemoryPool(GENERAL_POOL, generalPoolSize));
```
简单的说，
RESERVED_POOL大小由config.properties里的query.max-memory-per-node指定；
SYSTEM_POOL由config.properties里的resources.reserved-system-memory指定，如果不指定，默认值为Runtime.getRuntime().maxMemory() 0.4，即0.4 Xmx值。而GENERAL_POOL值为：
总内存（Xmx值）- 预留的（max-memory-per-node）- 系统的（0.4 * Xmx）。

从Presto的开发手册中可以看到：

```
GENERAL_POOL is the memory pool used by the physical operators in a query.
SYSTEM_POOL is mostly used by the exchange buffers and readers/writers.
RESERVED_POOL is for running a large query when the general pool becomes full.
```

简单说GENERAL_POOL用于普通查询的physical operators；SYSTEM_POOL用于读写buffer；而RESERVED_POOL比较特殊，大部分时间里是不参与计算的，只有当同时满足如下情形下，才会被使用，然后从所有查询里获取占用内存最大的那个查询，然后将该查询放到 RESERVED_POOL 里执行，同时注意RESERVED_POOL只能用于一个Query。

我们经常遇到的几个错误：

```
Query exceeded per-node total memory limit of xx
适当增加query.max-total-memory-per-node。

Query exceeded distributed user memory limit of xx
适当增加query.max-memory。

Could not communicate with the remote task. The node may have crashed or be under too much load
内存不够，导致节点crash，可以查看/var/log/message。
```

**并行度**
调整线程数增大 task 的并发以提高效率。
修改参数
![](https://github.com/chenxh/interviews/raw/main/imgs/presto-param.png "")


**SQL优化**

* 只选择使用必要的字段：由于采用列式存储，选择需要的字段可加快字段的读取、减少数据量。避免采用 * 读取所有字段

* 过滤条件必须加上分区字段

* Group By语句优化：合理安排Group by语句中字段顺序对性能有一定提升。将Group By语句中字段按照每个字段distinct数据多少进行降序排列， 减少GROUP BY语句后面的排序一句字段的数量能减少内存的使用.

* Order by时使用Limit， 尽量避免ORDER BY：Order by需要扫描数据到单个worker节点进行排序，导致单个worker需要大量内存

* 使用近似聚合函数：对于允许有少量误差的查询场景，使用这些函数对查询性能有大幅提升。比如使用approx_distinct() 函数比Count(distinct x)有大概2.3%的误差

* 用regexp_like代替多个like语句：Presto查询优化器没有对多个like语句进行优化，使用regexp_like对性能有较大提升

* 使用Join语句时将大表放在左边：Presto中join的默认算法是broadcast join，即将join左边的表分割到多个worker，然后将join右边的表数据整个复制一份发送到每个worker进行计算。如果右边的表数据量太大，则可能会报内存溢出错误。

* 使用Rank函数代替row_number函数来获取Top N

* UNION ALL 代替 UNION ：不用去重

* 使用WITH语句：查询语句非常复杂或者有多层嵌套的子查询，请试着用WITH语句将子查询分离出来


**元数据缓存**
Presto 支持 Hive connector，元数据存储在 Hive metastore 中，调整元数据缓存的相关参数可以提高访问元数据的效率。

**Hash 优化**
针对 Hash 场景的优化

**优化 OBS 相关参数**
Presto 支持 on OBS，读写 OBS 过程中可以调整 OBS 客户端参数来提交读写效率。


## Presto数据模型

Presto采取了三层表结构，我们可以和Mysql做一下类比：

catalog 对应某一类数据源，例如hive的数据，或mysql的数据

schema 对应mysql中的数据库

table 对应mysql中的表


## Presto为什么这么快？

* 完全基于内存的并行计算

* 流水线式的处理

* 本地化计算

* 动态编译执行计划

* 小心使用内存和数据结构

* 类BlinkDB的近似查询

* GC控制

和Hive这种需要调度生成计划且需要中间落盘的核心优势在于：Presto是常驻任务，接受请求立即执行，全内存并行计算；Hive需要用yarn做资源调度，接受查询需要先申请资源，启动进程，并且中间结果会经过磁盘。

