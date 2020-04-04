<!-- toc -->
## Coordinator进程
### 配置
对于Apache Druid的Coordinator进程配置，详见 [Coordinator配置]()

### HTTP
对于Coordinator的API接口，详见 [Coordinator API]()

### 综述
Druid Coordinator程序主要负责段管理和分发。更具体地说，Druid Coordinator进程与Historical进程通信，根据配置加载或删除段。Druid Coordinator负责加载新段、删除过时段、管理段复制和平衡段负载。

Druid Coordinator定期运行，每次运行之间的时间是一个可配置的参数。每次运行Druid Coordinator时，它都会在决定要采取的适当操作之前评估集群的当前状态。与Broker和Historical进程类似，Druid Coordinator维护了一个用于当前集群信息的Zookeeper集群连接。Coordinator还维护到数据库的连接，该数据库包含有关可用段和规则的信息。可用段存储在段表中，并列出应加载到集群中的所有段。规则存储在规则表中，并指示应如何处理段。

在Historical进程为任何未分配的段提供服务之前，首先按容量对每个层的可用Historical进程进行排序，最小容量的服务器具有最高优先级。未分配的段总是分配给具有最小能力的进程，以保持进程之间的平衡级别。在为Historical进程分配新段时，Coordinator不直接与该进程通信；而是在ZK中的Historical进程加载队列路径下创建有关该新段的一些临时信息。一旦看到此请求，Historical进程将加载段并开始为其提供服务。

### 运行
```
org.apache.druid.cli.Main server coordinator
```
### 规则
可以根据一组规则自动从集群中加载和删除段。有关规则的详细信息，请参阅[规则配置]()。

### 清理段
每次运行时，Druid Coordinator都会将数据库中可用段的列表与集群中的当前段进行比较,不在数据库中但仍在集群中服务的段将被标记并附加到删除列表中,被遮蔽的段（它们的版本太旧，它们的数据被更新的段替换）也会被丢弃。

### 段可用性
如果一个Historical进程由于任何原因重新启动或变得不可用，Druid Coordinator将注意到一个Historical进程已经丢失，并将该进程服务的所有段视为被删除。给定足够的时间后，这些段可以重新分配给集群中的其他Historical进程。然而，每个被删除的片段并不会立即被遗忘，取而代之的是一种过渡数据结构，它存储所有丢弃的具有相关生存期的段。生存期表示Coordinator不会重新分配丢弃的段的一段时间。因此，如果某个Historical进程在短时间内变得不可用并再次可用，则该Historical进程将启动并从其缓存中服务段，而不会在集群中重新分配任何这些段。

### 平衡段负载
为了确保在集群中跨Historical进程均匀分布段，Coordinator进程将在每次进程运行时查找每个Historical进程所服务的所有段的总大小。对于集群中的每个Historical进程层，Coordinator进程将确定利用率最高的Historical进程和利用率最低的Historical进程。计算两个进程之间利用率的百分比差异，如果结果超过某个阈值，则会将多个段从利用率最高的进程移动到利用率最低的进程。每次运行Coordinator时，可以从一个进程移动到另一个进程的段数有一个可配置的限制。要移动的段是随机选择的，只有在计算结果表明最高服务器和最低服务器之间的百分比差异减小时才会移动。

### 合并段
### 段查找策略
### Coordinator控制台
### FAQ