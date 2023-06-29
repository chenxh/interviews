## 设计目标
### 平台无关性
支持多种硬件和操作系统平台。 例如：RHEL, SLES, Ubuntu, Windows 等。平台上的组件（例如 yum， dep 包）应该是可拔插的，并且具有良好的定义接口。

### 可拔插的组件
整体架构不会采用指定的工具和技术。任何特定的工具和技术必须封装为可拔插的组件。
架构设计主要关注puppet 和 配置管理工具的可拔插性，以及使用数据库持久化状态。
目标不是立即替换puppet，而是在将来需要的时候可以轻松扩展。
可拔插性设计的范围不包括组件间协议或与三方组件的接口的标准化。

### 版本管理与升级
Ambari 被设计为可以在多种节点上运行，为了支持组件独立升级版本，必须支持多版本。
Ambari的任何组件升级不能影响集群状态。

### 可扩展性
Ambari 支持新增 service，components 和 API 。
可扩展性还意味着易于修改任何 Hadoop stack 的配置或部署步骤。Hadoop stack 是指以 Hadoop 为基础的做数据处理的一系列组件， 必须是完整的、且通过测试的组合， 例如：Cloudera的CDH， Ambari 默认带的HDP等。Ambari 还支持HDP之外的 hadoop stack 。

### 故障恢复
系统必须能够从任何组件故障恢复到一致状态。能在恢复后尝试完成待处理的操作。如果某些错误是不可恢复的，出现故障时仍应保持系统的一致性状态。

### 安全性
安全性意味着
1) Ambari的身份验证和基于角色的用户授权(包括API和Web UI) 。
2)通过Kerberos安装、管理和监控保护Hadoop栈;
3) Ambari 组件间的通信(例如Ambari master-agent)之间的进行身份验证和加密。

### 异常追踪
尽量简化异常追踪的过程。所有的失败都应该传导给用户，并且给用户足够的详细信息和关键点用以分析。

### 操作的近实时和中间状态的反馈
对于需要一段时间才能完成的操作，系统需要能够及时的提供给用户当前运行任务的进度反馈，例如任务完成的百分比，参考操作日志等。
在之前的Ambari版本中，由于Puppet的Master-Agent架构及其状态报告机制，并没有实现上述反馈。


## 术语
### Service（服务）
Service 指的是Hadoop stack 定义中的 services。  HDFS 、HBase 和 Pig 都是Service。 一个 Service 可能包含一个或者多个组件（例如：HDFS 包含 NameNode、 Secondary NameNode、DataNode 等）。一个 Service 也可以只是一个客户端库（例如：Pig没有任何后端进程，只有客户端库）

### Component （组件）
一个 Service 包含一个或者多个 Component 。例如：HDFS 包含 NameNode、 Secondary NameNode、DataNode 等。一个 Component 可以跨越多个主机节点（Node），例如 DataNode 可能在多个主机上部署。

### Node/Host （节点/服务器）
Node 指的是集群中的一台服务器。Node 和 Host 在本文档中可以互换。

### Node-Component
Node-Component 指的是一台机器上一个组件（Component）。

### Operation(操作)
操作就是集群上为了满足用户操作或者达到期望状态改变而进行一系列的系统改变和动作（Action）的集合。
例如：启动一个 Service 是一个操作，进行一次 smoke test（冒烟测试） 也是一个操作。
如果用户添加了一个新的 Service 到集群， 这个也会包含运行 somke test， 那么为了满足用户操作的，这些所有的系统行为（包含 smoke test） 组成了Operation。
一次Operation 包含了多次有序的系统动作（Action）。

### Task(任务)
任务是发送到节点上运行的工作单元。 任务是动作的一部分，一个节点必须执行的任务。
比如：一个动作包含在节点n1上安装 DataNode 和在节点 n2 上安装 DataNode 和 从 NameNode 。 那么 n1 上的 task 安装 DataNode, n2 上的 task 是安装 DataNode 和 NameNode。 

### Stage（阶段）
一个 Stage 包含了要完成某个操作的多个 Task， 每个 Stage 中的 Task 都是独立的。同一个 Stage 中的所以 Tasks 都可以并行运行。

### Action（动作）
一个 Action(动作) 包含了一台机器或者一组机器上的一个或者多个任务。每个动作可以用action id 来跟踪，节点会定时报送动作状态。
一个动作可以认为是一次执行一个阶段 Stage 。本文档中阶段和动作可以一一对应，个别特例除外。一个 action id 是 request id 和 stage-id 的组合。

### Stage Plan （阶段计划）
一个操作一般有多个在不同机器上运行任务，这些任务之间有依赖，这就需要按顺序运行。有些任务被调度之前需要先完成一些任务。
因此这些任务可以划分未不同的阶段Stage ，Stage 有顺序， 下一个 Stage 必须在前一个 Stage 完成后执行。每个 Stage 中的所有任务都可以并行运行。

### Manifest （清单）
Manifest 定义了发送到节点上运行的任务。Manifest 完整的定义一个任务，并且能够被序列化。Manifest 也能被持久化，用于恢复或者记录。

### Role （角色）
一个角色映射到一个 component （eg：DataNode）或者一个动作（eg: HDFS 执行重新均衡 等管理命令）。

## Ambari 的架构
Ambari 的总体框架如下：
![](https://github.com/chenxh/interviews/blob/main/imgs/ambari-0.jpg  "ambari-0")


Ambari Server 的设计
![](https://github.com/chenxh/interviews/blob/main/imgs/ambari-server.jpg  "ambari-server")


Ambari Agent 的设计
![](https://github.com/chenxh/interviews/blob/main/imgs/ambari-agent.jpg  "ambari-agent")

## 用例
### 添加 Service
在集群中添加一个新的 Service 。 以 HBase 为例， 假定集群已经部署了 HDFS 。在一些节点上部署 HBase 的 master 和 slave 。Ambari 的运行步骤如下：
*  请求通过 API 模块（ServiceService.createService）到达服务器并生成请求id 并且把id附在请求上。在协调器（coordinator）中调用此API的处理程序（handler）。 
* API处理程序实现启动新服务到集群所需的步骤。本用例中的步骤为：安装所有服务组件以及所需的依赖组件，按特定顺序排列启动前置组件和服务组件，并重新配置Nagios服务器，同时添加新的服务监控。
***Ambari 使用 Nagios 进行组件监控，所以有新的Service ，需要更新 Nagios 配置，添加监控***
* 协调器通过 Dependency Tracker 查看 HBase 的依赖。本例中 Dependency Tracker  会查找到 HBase 依赖 HDFS 和 ZooKeeper 组件。
协调器还会查找 Nagios Server 的依赖，本例中就是 HBase Client 。
协调器也会返回需要组件的状态，所以知道的所有需要的组件和它们的状态。协调器将在数据库中设置所有这些组件的期望状态。

***最新代码的协调器是在ServiceResourceProvider 中实现***

* 在前一个步骤中，协调器还可以确定需要用户的输入来为ZooKeeper选择节点，并可能返回一个合适的响应。这取决于API语义。
* 然后，协调器将传递组件及其所需状态的列表给 Stage Planner。Stage Planner 将返回的 Stage 列表， 这些 Stage 需要在这些组件所在的每个节点上执行的操作
安装、启动或修改。Stage Planner 也会生成使用 Manifest 文件列出(每个阶段中每个单独节点的任务)。
* 协调器将此有序的 Stage 列表和对应的请求 ID 传递给 ActionManager 。
* Action Manager将更新FSM每个node-component的状态，表示一项行动正在进行中。注意，每个受影响的节点组件的FSM都会更新。在这一步中，FSM可能检测到一个无效事件并抛出失败，然后中止操作，所有操作都将标记为失败并产生错误。
* Action Manager 将为每个操作创建一个行动 id，并将其添加到计划中。Action Manager 将从计划中选择第一阶段，并将此阶段中的每个操作添加到每个受影响节点的队列中。当第一阶段完成时，将选择下一阶段。Action Manager 还将为预定的操作启动一个计时器。
* Heartbeat Handler 将接收操作的响应并通知 Action Manager 。Action Manager会向 FSM 发送一个事件来更新状态。在超时的情况下，该操作将再次被调度或标记为失败。一旦某个操作的所有节点都完成(收到响应或最终超时)，该操作就被认为完成。一旦某个阶段的所有操作完成，该阶段就被认为完成了。
* 操作完成记录在数据库中。
*  Action Manager 继续进行下一个 Stage， 以同样方式运行后续 Stage

### 运行冒烟测试
假设集群已经存在 HDFS 和 HBase 服务。用户运行 HBase 冒烟测试。
* 请求通过 API 模块到达服务器并生成请求id 并且把id附在请求上。在协调器（coordinator）中调用此API的处理程序（handler）。
* API处理程序调用 Dependency Tracker ，发现HBase服务已经启动并运行。如果hbase的 live 状态没有运行，API处理程序会抛出错误。处理程序还确定在哪里运行冒烟测试。Dependency Tracker 会告诉协调器在需要运行冒烟测试的主机上需要哪个客户端组件。Stage Planner 用于生成发烟测试的计划。在这种情况下，计划很简单，只有一个节点组件的 Stage。
* 其余步骤与上面的用例类似。在这种情况下，FSM将专门用于hbase-smoketest角色 。

### 重新配置 Service
集群中的服务及其依赖组件已经处于活动状态。
* 用户保存配置，新的配置快照存储在服务器上。该请求还应该包含配置更改影响的服务和/或角色的信息。这意味着持久层需要跟踪每个服务/组件/节点上影响所述对象的最后一个更改的配置版本。
* 用户可以通过以上多次呼叫来保存多个检查点。
* 在某个时候，用户决定部署新配置。在这个场景中，用户将发送一个请求，将新的配置部署到所需的服务/组件/节点组件。
* 当这个请求通过协调器处理程序到达服务器时，它将导致相关对象的所需配置版本更新为指定的版本或最新版本。
* 协调器将分两步执行re-configure。首先，它将生成一个Stage 计划，停止相关服务。然后，生成一个 Stage 计划，使用新配置启动服务。协调器将添加两个计划，stop和start，并将其传递给 Action Manager。其余步骤按照前面的用例进行。

### Ambari Master 的崩溃和重启
* 如果 Ambari Master 宕机，有多种场景需要处理。
* 假设条件：
    * 期望状态已经持久化到DB了。
    * 所有挂起的操作都已经在数据库中定义了。
    * 现场状态也在数据库中有记录。但是，agent 当前不能发送现场状态到server 端，因此所有的需求将基于数据库状态。
    * 所有的动作（Action）都是幂等的。
* Master 重启需要执行的动作
    * Ambari Master
        * 动作层需要将所有待决或之前排队(但不完整)的阶段重新排队到心跳处理程序，并允许代理重新执行所述步骤。
        *  Ambari Server 在恢复前不再接受新的动作和请求。
        * 如果实际的现场状态和期望状态之间有任何差异（例如：实际状态时starting、started， 而期望状态时stopped），那么Stage Planner 或者 事务管理器应该有一个触发器来启动操作，以将状态转换到期望状态。但是，如果活动状态是STOP_FAILED, 期望状态是 STOPPED，那么这是一个空操作，因为在这种情况下，Ambari 服务器本身不会发起操作。对于后一种情况，管理员/用户负责为失败的节点重新启动停止。
    *  Ambari Agent
        * 如果Master 宕机，Agent 应该没有停止。Agent 会定时拉取状态， 在 Master 重启后会自动恢复。
        * Agent 在和 Master 的连接断掉后，会保存所有计划发送到Master 的必要信息， 并且在Master 启动后重试发送。 
        * 如果它之前在注册过程中，可能需要重新注册。
### 卸载部分 DataNode 节点
* 卸载将作为在Puppet层中对adoop admin角色执行的操作来实现。hadoop-admin将是一个新的角色，涵盖hadoop管理操作。这个角色需要安装hadoop-client并且有管理员权限。
*  卸载操作的 Manifest 文件由hadoop-admin角色和一些参数组成，比如一个datanode列表和一个指定操作的标志。
* 协调器将确定指定运行hadoop-admin操作的节点。此信息必须在集群状态中可用。
* 对于作为操作的节点角色，卸载操作也要按照状态机的机制运行。当卸载已经在namenode上启动，即datanode的admin命令执行成功时，则认为卸载成功。
* 节点是否已卸载信息需要单独查询。这些信息可以在namenode UI中找到。Ambari应该有另一个API来查询正在卸载或已经卸载的datanode。
* Ambari不会跟踪datanode是否正在卸载。要从Ambari中删除已卸载的节点，需要向Ambari单独调用API来停止/卸载这些 datanode。

## Agent
Agent 定时每隔几秒发送心跳到Master， Master 会把需要运行的命令放在心跳的响应中。心跳是Master 发送命令到 Agent 的唯一途径。 
在 Agent 端会把接受到的命令放到 action queue 中，Action executioner 从队列中取命令。
Action executioner 根据命令类型和动作（action）类型来选择工具（Puppet， Python 等）执行命令。
发送到 Agent 的命令是异步运行的。
Action executioner  把响应或者处理中信息放到 message queue 中。
Agent 会在下一次心跳中发送 message queue 中的消息。

## 恢复
Master 的恢复一般有两种选择：
* 基于动作(action) 来恢复:  这种方案中所有的动作（action）都要持久化，重启后，Master 检查待处理的动作（action），然后重新调度运行。集群状态也是持久化到数据库中， Master 重启后重新构建状态机。此方案的的条件比较苛刻，比如： 动作（action）实际执行完成了，但是在记录状态之前 Master 宕机了。 这种情况下，就要求动作（action）是幂等的，可重复运行的， Master 会重复调度这些动作。持久化的动作可以视作 redo log 。
* 基于期望状态恢复：Master 持久化期望状态，重启后比较期望状态和实际状态，并且重置集群状态为期望状态。
基于动作的方法更适合我们的整体设计，因为我们是预先计划一个操作并将其持久化，因此即使我们持久化了所需的状态，恢复也需要重新计划动作（Action）。此外，从Ambari的角度来看，基于期望状态的方式也不会捕获不会改变集群状态的操作，例如冒烟测试或hdfs重新平衡。持久化的操作可以被视为重做日志。

Agent 的恢复只需要重启，因为 Agent 是无状态的。Master 通过检测心跳是否在时限内丢失来判断 Agent 是否失败。

## 安全
### API 认证 
一种选择是使用 HTTP Basic 认证。这种方式客户端需要在每次的请求中传输凭证。服务端需要每次请求都验证凭证。对于命令行来说这种方式很友好。Session 也不需要存储在服务端。
另一种方式是使用 HTTP Session 和 Cookie 。 Server 端验证凭证后，生成 Sesssion 并存储。 Session ID 发送到客户端，并以cookie的方式存储。客户端请求时带上Session ID。这种方式效率比较高。VMware’s vCloud REST API 1.0 以前支持这种方式， 但是在新版本中去掉了，因为安全问题：http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displ
ayKC&externalId=1025884。而且服务端存储 Session 不太符合 RESTFul 规范。

Ambari 支持两种方式。 对于命令行，第一种方式更好。对于浏览器应用第二种方式避免了重复验证，更高效。
这两种方式，都基于 HTTPS 。

## 启动
Ambari 1.0之前启动是一个很精密的过程，包含安装和配置host。 在 1.1 版本中提供了帮组程序配置host和安装 Agent。
在1.0 中启动程序会计算出 host 的相关信息，然后保存到数据库。现在 Agent 在注册过程中会完成这个工作。启动程序只是用SSH 登录到机器上，然后安装Agent ， 不会修改数据库。













        



















 





