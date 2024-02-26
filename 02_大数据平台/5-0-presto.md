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

![](https://github.com/chenxh/interviews/blob/main/imgs/presto.png "")

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
![](https://github.com/chenxh/interviews/blob/main/imgs/presto-plan.png "")


