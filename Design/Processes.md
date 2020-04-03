<!-- toc -->

## 进程和服务
### 进程类型

Druid有以下几种进程类型：
* [Coordinator](Coordinator.md)
* [Overlord](Overlord.md)
* [Broker](Broker.md)
* [Historical](Historical.md)
* [MiddleManager](MiddleManager.md) 和 [Peons](Peons.md)
* [Indexer(可选)](Indexer.md)
* [Router(可选)](Router.md)

### 服务类型

Druid进程可以按照您喜欢的任何方式部署，但是为了便于部署，我们建议将它们组织成三种服务器类型：
* **Master**
* **Query**
* **Data**

![](img/druid-architecture.png)

本节描述Druid进程和建议的Master/Query/Data server组织，如上面的架构图所示。

#### Master服务

Master服务管理数据的摄取和可用性：它负责启动新的摄取作业并协调下面描述的"Data服务"上数据的可用性。

在Master服务中，功能分为两个进程：Coordinator和Overlord。

**Coordinator进程**

[Coordinator](Coordinator.md) 监视Data服务中的Historical进程，它们负责将数据段分配给特定的服务器，并确保数据段在各个Historical之间保持良好的平衡。

**Overlord进程**

[Overlord](Overlord.md) 监视Data服务中的MiddleManager进程，并且是Druid数据接收的控制器。它们负责将接收任务分配给MiddleManager，并协调数据段的发布。

#### Query服务

Query服务提供用户和客户端应用程序交互，将查询路由到Data服务或其他Query服务（以及可选的代理Master服务请求）。

在Query服务中，功能上分为两个进程：Broker和Router。

**Broker进程**

[Broker](Broker.md)从外部客户端接收查询并将这些查询转发到Data服务器, 当Broker接收到子查询的结果时，它们会合并这些结果并将其返回给调用者。用户通常查询Broker，而不是直接查询Data服务中的Historical或MiddleManager进程。

**Router进程（可选）**

Router进程是*可选*的进程，相当于是为Druid Broker、Overlord和Coordinator提供一个统一的API网关。Router是可选的，因为也可以直接与Druid的Broker、Overlord和Coordinator。

Router还运行着[Druid控制台]()，一个用于数据源、段、任务、数据进程（Historical和MiddleManager）和Coordinator动态配置的管理UI。用户还可以在控制台中运行SQL和本地Druid查询。

#### Data服务
#### Historical进程
#### MiddleManager进程
#### Indexer进程（可选）
### 服务托管的利弊
#### Coordinator和Overlord
#### Historical和MiddleManager