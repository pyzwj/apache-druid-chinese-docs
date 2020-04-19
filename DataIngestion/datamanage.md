<!-- toc -->
## 数据管理

### schema更新
数据源的schema可以随时更改，Apache Druid支持不同段之间的有不同的schema。
#### 替换段文件
Druid使用数据源、时间间隔、版本号和分区号唯一地标识段。只有在某个时间粒度内创建多个段时，分区号才在段id中可见。例如，如果有小时段，但一小时内的数据量超过单个段的容量，则可以在同一小时内创建多个段。这些段将共享相同的数据源、时间间隔和版本号，但具有线性增加的分区号。
```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-01/2015-01-02_v1_1
foo_2015-01-01/2015-01-02_v1_2
```
在上面的示例段中，`dataSource`=`foo`，`interval`=`2015-01-01/2015-01-02`，version=`v1`，partitionNum=`0`。如果在以后的某个时间点，使用新的schema重新索引数据，则新创建的段将具有更高的版本id。
```
foo_2015-01-01/2015-01-02_v2_0
foo_2015-01-01/2015-01-02_v2_1
foo_2015-01-01/2015-01-02_v2_2
```
Druid批索引(基于Hadoop或基于IndexTask)保证了时间间隔内的原子更新。在我们的例子中，直到 `2015-01-01/2015-01-02` 的所有 `v2` 段加载到Druid集群中之前，查询只使用 `v1` 段, 当加载完所有v2段并可查询后，所有查询都将忽略 `v1` 段并切换到 `v2` 段。不久之后，`v1` 段将从集群中卸载。

请注意，跨越多个段间隔的更新在每个间隔内都是原子的。在整个更新过程中它们不是原子的。例如，您有如下段：
```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v1_1
foo_2015-01-03/2015-01-04_v1_2
```
`v2` 段将在构建后立即加载到集群中，并在段重叠的时间段内替换 `v1` 段。在完全加载 `v2` 段之前，集群可能混合了 `v1` 和 `v2` 段。
```
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v2_1
foo_2015-01-03/2015-01-04_v1_2
```
在这种情况下，查询可能会命中 `v1` 和 `v2` 段的混合。
#### 在段中不同的schema
同一数据源的Druid段可能有不同的schema。如果一个字符串列（维度）存在于一个段中而不是另一个段中，则涉及这两个段的查询仍然有效。对缺少维度的段的查询将表现为该维度只有空值。类似地，如果一个段有一个数值列（metric），而另一个没有，那么查询缺少metric的段通常会"做正确的事情"。在此缺失的Metric上的使用聚合的行为类似于该Metric缺失。
### 压缩与重新索引
压缩合并是一种覆盖操作，它读取现有的一组段，将它们组合成一个具有较大但较少段的新集，并用新的压缩集覆盖原始集，而不更改它内部存储的数据。

出于性能原因，有时将一组段压缩为一组较大但较少的段是有益的，因为在接收和查询路径中都存在一些段处理和内存开销。

压缩任务合并给定时间间隔内的所有段。语法为：
```
{
    "type": "compact",
    "id": <task_id>,
    "dataSource": <task_datasource>,
    "ioConfig": <IO config>,
    "dimensionsSpec" <custom dimensionsSpec>,
    "metricsSpec" <custom metricsSpec>,
    "segmentGranularity": <segment granularity after compaction>,
    "tuningConfig" <parallel indexing task tuningConfig>,
    "context": <task context>
}
```

| 字段 | 描述 | 是否必须 |
|-|-|-|
| `type` | 任务类型，应该是 `compact` | 是 |
| `id` | 任务id | 否 |
| `dataSource` | 将被压缩合并的数据源名称 | 是 | 
| `ioConfig` | 压缩合并任务的 `ioConfig`, 详情见 [Compaction ioConfig](#压缩合并的IOConfig) | 是 |
| `dimensionsSpec` | 自定义 `dimensionsSpec`。压缩任务将使用此dimensionsSpec（如果存在），而不是生成dimensionsSpec。更多细节见下文。| 否 |
| `metricsSpec` | 自定义 `metricsSpec`。如果指定了压缩任务，则压缩任务将使用此metricsSpec，而不是生成一个metricsSpec。| 否 |
| `segmentGranularity` | 如果设置了此值，压缩合并任务将更改给定时间间隔内的段粒度。有关详细信息，请参阅 [granularitySpec](ingestion.md#granularityspec) 的 `segmentGranularity`。行为见下表。 | 否 |
| `tuningConfig` | [并行索引任务的tuningConfig](native.md#tuningConfig) | 否 |
| `context` | [任务的上下文](taskrefer.md#上下文参数) | 否 |

一个压缩合并任务的示例如下：
```
{
  "type" : "compact",
  "dataSource" : "wikipedia",
  "ioConfig" : {
    "type": "compact",
    "inputSpec": {
      "type": "interval",
      "interval": "2017-01-01/2018-01-01"
    }
  }
}
```

压缩任务读取时间间隔 `2017-01-01/2018-01-01` 的*所有分段*，并生成新分段。由于 `segmentGranularity` 为空，压缩后原始的段粒度将保持不变。要控制每个时间块的结果段数，可以设置 [`maxRowsPerSegment`](../Configuration/configuration.md#Coordinator) 或 [`numShards`](native.md#tuningconfig)。请注意，您可以同时运行多个压缩任务。例如，您可以每月运行12个compactionTasks，而不是一整年只运行一个任务。

压缩任务在内部生成 `index` 任务规范，用于使用某些固定参数执行的压缩工作。例如，它的 `inputSource` 始终是 [DruidInputSource](native.md#Druid输入源)，`dimensionsSpec` 和 `metricsSpec` 默认包含输入段的所有Dimensions和Metrics。

如果指定的时间间隔中没有加载数据段（或者指定的时间间隔为空），则压缩任务将以失败状态代码退出，而不执行任何操作。

除非所有输入段具有相同的元数据，否则输出段可以具有与输入段不同的元数据。

* Dimensions: 由于Apache Druid支持schema更改，因此即使是同一个数据源的一部分，各个段之间的维度也可能不同。如果输入段具有不同的维度，则输出段基本上包括输入段的所有维度。但是，即使输入段具有相同的维度集，维度顺序或维度的数据类型也可能不同。例如，某些维度的数据类型可以从 `字符串` 类型更改为基本类型，或者可以更改维度的顺序以获得更好的局部性。在这种情况下，在数据类型和排序方面，最近段的维度先于旧段的维度。这是因为最近的段更有可能具有所需的新顺序和数据类型。如果要使用自己的顺序和类型，可以在压缩任务规范中指定自定义 `dimensionsSpec`。
* Roll-up: 仅当为所有输入段设置了 `rollup` 时，才会汇总输出段。有关详细信息，请参见 [rollup](ingestion.md#rollup)。您可以使用 [段元数据查询](../Querying/segmentMetadata.md) 检查段是否已被rollup。


#### 压缩合并的IOConfig
### 增加新的数据
### 更新现有的数据
#### 使用lookups
#### 重新摄取数据
#### 使用基于Hadoop的摄取
#### 使用原生批摄取重新索引
### 删除数据
### 杀掉任务
### 数据保留