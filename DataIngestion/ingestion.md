<!-- toc -->

## 数据摄入
### 综述

Druid中的所有数据都被组织成*段*，这些段是数据文件，通常每个段最多有几百万行。在Druid中加载数据称为*摄取或索引*，它包括从源系统读取数据并基于该数据创建段。

在大多数摄取方法中，加载数据的工作由Druid [MiddleManager](../Design/MiddleManager.md) 进程（或 [Indexer](../Design/Indexer.md) 进程）完成。一个例外是基于Hadoop的摄取，这项工作是使用Hadoop MapReduce作业在YARN上完成的（尽管MiddleManager或Indexer进程仍然参与启动和监视Hadoop作业）。一旦段被生成并存储在 [深层存储](../Design/Deepstorage.md) 中，它们将被Historical进程加载。有关如何在引擎下工作的更多细节，请参阅Druid设计文档的[存储设计](../Design/Design.md) 部分。

### 如何使用本文档

您**当前正在阅读的这个页面**提供了通用Druid摄取概念的信息，以及 [所有摄取方法](#摄入方式) **通用的配置**信息。

**每个摄取方法的单独页面**提供了有关每个摄取方法**独有的概念和配置**的附加信息。

我们建议您先阅读（或至少略读）这个通用页面，然后参考您选择的一种或多种摄取方法的页面。

### 摄入方式

下表列出了Druid最常用的数据摄取方法，帮助您根据自己的情况选择最佳方法。每个摄取方法都支持自己的一组源系统。有关每个方法如何工作的详细信息以及特定于该方法的配置属性，请查看其文档页。

#### 流式摄取
最推荐、也是最流行的流式摄取方法是直接从Kafka读取数据的 [Kafka索引服务](kafka.md) 。如果你喜欢Kinesis，[Kinesis索引服务](kinesis.md) 也能很好地工作。

下表比较了主要可用选项：

| **Method** | [**Kafka**](kafka.md) | [**Kinesis**](kinesis.md) | [**Tranquility**](tranquility.md) |
| - | - | - | - |
| **Supervisor类型** | `kafka` | `kinesis` | `N/A` |
| 如何工作 | Druid直接从 Apache Kafka读取数据 | Druid直接从Amazon Kinesis中读取数据 | Tranquility, 一个独立于Druid的库，用来将数据推送到Druid |
| 可以摄入迟到的数据 | Yes | Yes | No(迟到的数据将会被基于 `windowPeriod` 的配置丢弃掉) |
| 保证不重不丢（Exactly-once）| Yes | Yes | No

#### 批量摄取

从文件进行批加载时，应使用一次性 [任务](taskrefer.md)，并且有三个选项：`index_parallel`（本地并行批任务）、`index_hadoop`（基于hadoop）或`index`（本地简单批任务）。

一般来说，如果本地批处理能满足您的需要时我们建议使用它，因为设置更简单（它不依赖于外部Hadoop集群）。但是，仍有一些情况下，基于Hadoop的批摄取可能是更好的选择，例如，当您已经有一个正在运行的Hadoop集群，并且希望使用现有集群的集群资源进行批摄取时。

此表比较了三个可用选项：

| **方式** | [**本地批任务(并行)**](native.md#并行任务) | [**基于Hadoop**](hadoopbased.md) | [**本地批任务(简单)**](native.md#简单任务) |
| - | - | - | - |
| **任务类型** | `index_parallel` | `index_hadoop` | `index` |
| **并行?** | 如果 `inputFormat` 是可分割的且 `tuningConfig` 中的 `maxNumConcurrentSubTasks` > 1, 则 **Yes** | Yes | No，每个任务都是单线程的 |
| **支持追加或者覆盖** | 都支持 | 只支持覆盖 | 都支持 |
| **外部依赖** | 无 | Hadoop集群，用来提交Map-Reduce任务 | 无 |
| **输入位置** | 任何 [输入数据源](native.md#输入数据源) | 任何Hadoop文件系统或者Druid数据源 |  任何 [输入数据源](native.md#输入数据源) |
| **文件格式** | 任何 [输入格式](dataformats.md) | 任何Hadoop输入格式 | 任何 [输入格式](dataformats.md) |
| [**Rollup modes**](#Rollup) | 如果 `tuningConfig` 中的 `forceGuaranteedRollup` = true, 则为 **Perfect(最佳rollup)** | 总是Perfect（最佳rollup） | 如果 `tuningConfig` 中的 `forceGuaranteedRollup` = true, 则为 **Perfect(最佳rollup)** |
| **分区选项** | 可选的有`Dynamic`, `hash-based` 和 `range-based` 三种分区方式，详情参见 [分区规范](native.md#partitionsSpec) | 通过 [partitionsSpec](hadoopbased.md#partitionsSpec)中指定 `hash-based` 和 `range-based`分区 | 可选的有`Dynamic`和`hash-based`二种分区方式，详情参见 [分区规范](native.md#partitionsSpec) |

### Druid数据模型
#### 数据源
Druid数据存储在数据源中，与传统RDBMS中的表类似。Druid提供了一个独特的数据建模系统，它与关系模型和时间序列模型都具有相似性。
#### 主时间戳列
Druid Schema必须始终包含一个主时间戳。主时间戳用于对 [数据进行分区和排序](#分区)。Druid查询能够快速识别和检索与主时间戳列的时间范围相对应的数据。Druid还可以将主时间戳列用于基于时间的[数据管理操作](datamanage.md)，例如删除时间块、覆盖时间块和基于时间的保留规则。

主时间戳基于 [`timestampSpec`](#timestampSpec) 进行解析。此外，[`granularitySpec`](#granularitySpec) 控制基于主时间戳的其他重要操作。无论从哪个输入字段读取主时间戳，它都将作为名为 `__time` 的列存储在Druid数据源中。

如果有多个时间戳列，则可以将其他列存储为 [辅助时间戳](schemadesign.md#辅助时间戳)。

#### 维度
维度是按原样存储的列，可以用于任何目的, 可以在查询时以特殊方式对维度进行分组、筛选或应用聚合器。如果在禁用了 [rollup](#Rollup) 的情况下运行，那么该维度集将被简单地视为要摄取的一组列，并且其行为与不支持rollup功能的典型数据库的预期完全相同。

通过 [`dimensionSpec`](#dimensionSpec) 配置维度。

#### 指标
Metrics是以聚合形式存储的列。启用 [rollup](#Rollup) 时，它们最有用。指定一个Metric允许您为Druid选择一个聚合函数，以便在摄取期间应用于每一行。这有两个好处：

1. 如果启用了 [rollup](#Rollup)，即使保留摘要信息，也可以将多行折叠为一行。在 [Rollup教程](../Tutorials/chapter-5.md) 中，这用于将netflow数据折叠为每（`minute`，`srcIP`，`dstIP`）元组一行，同时保留有关总数据包和字节计数的聚合信息。
2. 一些聚合器，特别是近似聚合器，即使在非汇总数据上，如果在接收时部分计算，也可以在查询时更快地计算它们。

Metrics是通过 [`metricsSpec`](#metricsSpec) 配置的。

### Rollup
#### 什么是rollup
Druid可以在接收过程中将数据进行汇总，以最小化需要存储的原始数据量。Rollup是一种汇总或预聚合的形式。实际上，Rollup可以极大地减少需要存储的数据的大小，从而潜在地减少行数的数量级。这种存储量的减少是有代价的：当我们汇总数据时，我们就失去了查询单个事件的能力。

禁用rollup时，Druid将按原样加载每一行，而不进行任何形式的预聚合。此模式类似于您对不支持汇总功能的典型数据库的期望。

如果启用了rollup，那么任何具有相同[维度](#维度)和[时间戳](#主时间戳列)的行（在基于 `queryGranularity` 的截断之后）都可以在Druid中折叠或汇总为一行。

rollup默认是启用状态。

#### 启用或者禁用rollup

Rollup由 `granularitySpec` 中的 `rollup` 配置项控制。 默认情况下，值为 `true`(启用状态)。如果你想让Druid按原样存储每条记录，而不需要任何汇总，将该值设置为 `false`。

#### rollup示例
有关如何配置Rollup以及该特性将如何修改数据的示例，请参阅[Rollup教程](../Tutorials/chapter-5.md)。

#### 最大化rollup比率
通过比较Druid中的行数和接收的事件数，可以测量数据源的汇总率。这个数字越高，从汇总中获得的好处就越多。一种方法是使用[Druid SQL](../Querying/druidsql.md)查询，比如：
```
SELECT SUM("cnt") / COUNT(*) * 1.0 FROM datasource
```
#### 最佳rollup VS 尽可能rollup
### 分区
#### 为什么分区
#### 如何设置分区
### 摄入规范
#### `dataSchema`
##### `dataSource`
##### `timestampSpec`
##### `dimensionSpec`
##### `metricsSpec`
##### `granularitySpec`
##### `transformSpec`
##### 过时的 `dataSchema` 规范
#### `ioConfig`
#### `tuningConfig`