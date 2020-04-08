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

在这个查询中，`cnt` 应该引用在摄取时指定的"count"类型Metrics。有关启用汇总时计数工作方式的详细信息，请参阅"架构设计"页上的 [计数接收事件数](../DataIngestion/schemadesign.md#计数接收事件数)。

最大化Rollup的提示：
* 一般来说，拥有的维度越少，维度的基数越低，您将获得更好的汇总比率
* 使用 [Sketches](schemadesign.md#Sketches高基维处理) 避免存储高基数维度，因为会损害汇总比率
* 在摄入时调整 `queryGranularity`（例如，使用 `PT5M` 而不是 `PT1M` ）会增加Druid中两行具有匹配时间戳的可能性，并可以提高汇总率
* 将相同的数据加载到多个Druid数据源中是有益的。有些用户选择创建禁用汇总（或启用汇总，但汇总比率最小）的"完整"数据源和具有较少维度和较高汇总比率的"缩写"数据源。当查询只涉及"缩写"集里边的维度时，使用该数据源将导致更快的查询时间，这种方案只需稍微增加存储空间即可完成，因为简化的数据源往往要小得多。
* 如果您使用的 [尽力而为的汇总(best-effort rollup)](#) 摄取配置不能保证[完全汇总(perfect rollup)](#)，则可以通过切换到保证的完全汇总选项，或在初始摄取后在[后台重新编制(reindex)](./datamanage.md#压缩与重新索引)数据索引，潜在地提高汇总比率。

#### 最佳rollup VS 尽可能rollup
一些Druid摄取方法保证了*完美的汇总(perfect rollup)*，这意味着输入数据在摄取时被完美地聚合。另一些则提供了*尽力而为的汇总(best-effort rollup)*，这意味着输入数据可能无法完全聚合，因此可能有多个段保存具有相同时间戳和维度值的行。

一般来说，提供*尽力而为的汇总(best-effort rollup)*的摄取方法之所以这样做，是因为它们要么是在没有清洗步骤（这是*完美的汇总(perfect rollup)*所必需的）的情况下并行摄取，要么是因为它们在接收到某个时间段的所有数据（我们称之为*增量发布(incremental publishing)*）之前完成并发布段。在这两种情况下，理论上可以汇总的记录可能会以不同的段结束。所有类型的流接收都在此模式下运行。

保证*完美的汇总(perfect rollup)*的摄取方法通过额外的预处理步骤来确定实际数据摄取阶段之前的间隔和分区。此预处理步骤扫描整个输入数据集，这通常会增加摄取所需的时间，但提供完美汇总所需的信息。

下表显示了每个方法如何处理汇总：
| **方法** | **如何工作** |
| - | - |
| [本地批](native.md) | 基于配置，`index_parallel` 和 `index` 可以是完美的，也可以是最佳的。 |
| [Hadoop批](hadoopbased.md) | 总是 perfect |
| [Kafka索引服务](kafka.md) | 总是 best-effort | 
| [Kinesis索引服务](kinesis.md) | 总是 best-effort |

### 分区
#### 为什么分区

数据源中段的最佳分区和排序会对占用空间和性能产生重大影响
。
Druid数据源总是按时间划分为*时间块*，每个时间块包含一个或多个段。此分区适用于所有摄取方法，并基于摄取规范的 `dataSchema` 中的 `segmentGranularity`参数。

特定时间块内的段也可以进一步分区，使用的选项根据您选择的摄取类型而不同。一般来说，使用特定维度执行此辅助分区将改善局部性，这意味着具有该维度相同值的行存储在一起，并且可以快速访问。

通常，通过将数据分区到一些常用来做过滤操作的维度（如果存在的话）上，可以获得最佳性能和最小的总体占用空间。而且，这种分区通常会改善压缩性能而且还往往会提高查询性能（用户报告存储容量减少了三倍）。

> [!WARNING]
> 分区和排序是最好的朋友！如果您确实有一个天然的分区维度，那么您还应该考虑将它放在 `dimensionsSpec` 的 `dimension` 列表中的第一个维度，它告诉Druid按照该列对每个段中的行进行排序。除了单独分区所获得的改进之外，这通常还会进一步改进压缩。
> 但是，请注意，目前，Druid总是首先按时间戳对一个段内的行进行排序，甚至在 `dimensionsSpec` 中列出的第一个维度之前，这将使得维度排序达不到最大效率。如果需要，可以通过在 `granularitySpec` 中将 `queryGranularity` 设置为等于 `segmentGranularity` 的值来解决此限制，这将把段内的所有时间戳设置为相同的值，并将"真实"时间戳保存为[辅助时间戳](./schemadesign.md#辅助时间戳)。这个限制可能在Druid的未来版本中被移除。

#### 如何设置分区
并不是所有的摄入方式都支持显式的分区配置，也不是所有的方法都具有同样的灵活性。在当前的Druid版本中，如果您是通过一个不太灵活的方法（如Kafka）进行初始摄取，那么您可以使用 [重新索引的技术(reindex)](./datamanage.md#压缩与重新索引)，在最初摄取数据后对其重新分区。这是一种强大的技术：即使您不断地从流中添加新数据, 也可以使用它来确保任何早于某个阈值的数据都得到最佳分区。

下表显示了每个摄取方法如何处理分区：

| **方法** | **如何工作** |
| - | - |
| [本地批](native.md) | 通过 `tuningConfig` 中的 [`partitionsSpec`](./native.md#partitionsSpec) |
| [Hadoop批](hadoopbased.md) | 通过 `tuningConfig` 中的 [`partitionsSpec`](./native.md#partitionsSpec)  |
| [Kafka索引服务](kafka.md) | Druid中的分区是由Kafka主题的分区方式决定的。您可以在初次摄入后 [重新索引的技术(reindex)](./datamanage.md#压缩与重新索引)以重新分区 | 
| [Kinesis索引服务](kinesis.md) | Druid中的分区是由Kinesis流的分区方式决定的。您可以在初次摄入后 [重新索引的技术(reindex)](./datamanage.md#压缩与重新索引)以重新分区 |

> [!WARNING]
> 
> 注意，当然，划分数据的一种方法是将其加载到分开的数据源中。这是一种完全可行的方法，当数据源的数量不会导致每个数据源的开销过大时，它可以很好地工作。如果使用这种方法，那么可以忽略这一部分，因为这部分描述了如何在单个数据源中设置分区。
> 
> 有关将数据拆分为单独数据源的详细信息以及潜在的操作注意事项，请参阅 [多租户注意事项](../Querying/multitenancy.md)。

### 摄入规范

无论使用哪一种摄入方式，数据要么是通过一次性[tasks](taskrefer.md)或者通过持续性的"supervisor"(运行并监控一段时间内的一系列任务)来被加载到Druid中。 在任一种情况下，task或者supervisor的定义都在*摄入规范*中定义。

摄入规范包括以下三个主要的部分：
* [`dataSchema`](#dataschema), 包含了 [`数据源名称`](#datasource), [`主时间戳列`](#timestampspec), [`维度`](#dimensionspec), [`指标`](#metricsspec) 和 [`转换与过滤`](#transformspec)
* [`ioConfig`](#ioconfig), 该部分告诉Druid如何去连接数据源系统以及如何去解析数据。 更多详细信息，可以看[摄入方法](#摄入方式)的文档。
* [`tuningConfig`](#tuningconfig), 该部分控制着每一种[摄入方法](#摄入方式)的不同的特定调整参数

一个 `index_parallel` 类型任务的示例摄入规范如下：
```
{
  "type": "index_parallel",
  "spec": {
    "dataSchema": {
      "dataSource": "wikipedia",
      "timestampSpec": {
        "column": "timestamp",
        "format": "auto"
      },
      "dimensionsSpec": {
        "dimensions": [
          { "type": "string", "page" },
          { "type": "string", "language" },
          { "type": "long", "name": "userId" }
        ]
      },
      "metricsSpec": [
        { "type": "count", "name": "count" },
        { "type": "doubleSum", "name": "bytes_added_sum", "fieldName": "bytes_added" },
        { "type": "doubleSum", "name": "bytes_deleted_sum", "fieldName": "bytes_deleted" }
      ],
      "granularitySpec": {
        "segmentGranularity": "day",
        "queryGranularity": "none",
        "intervals": [
          "2013-08-31/2013-09-01"
        ]
      }
    },
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "local",
        "baseDir": "examples/indexing/",
        "filter": "wikipedia_data.json"
      },
      "inputFormat": {
        "type": "json",
        "flattenSpec": {
          "useFieldDiscovery": true,
          "fields": [
            { "type": "path", "name": "userId", "expr": "$.user.id" }
          ]
        }
      }
    },
    "tuningConfig": {
      "type": "index_parallel"
    }
  }
}
```

该部分中支持的特定选项依赖于选择的[摄入方法](#摄入方式)。 更多的示例，可以参考每一种[摄入方法](#摄入方式)的文档。

您还可以不用编写一个摄入规范，可视化的加载数据，该功能位于 [Druid控制台](../Operations/manageui.md) 的 "Load Data" 视图中。 Druid可视化数据加载器目前支持 [Kafka](kafka.md), [Kinesis](kinesis.md) 和 [本地批](native.md) 模式。

#### `dataSchema`
> [!WARNING]
> 
> `dataSchema` 规范在0.17.0版本中做了更改，新的规范支持除*Hadoop摄取方式*外的所有方式。 可以在 [过时的 `dataSchema` 规范]()查看老的规范

`dataSchema` 包含了以下部分：
* [`数据源名称`](#datasource), [`主时间戳列`](#timestampspec), [`维度`](#dimensionspec), [`指标`](#metricsspec) 和 [`转换与过滤`](#transformspec)

一个 `dataSchema` 如下：
```
"dataSchema": {
  "dataSource": "wikipedia",
  "timestampSpec": {
    "column": "timestamp",
    "format": "auto"
  },
  "dimensionsSpec": {
    "dimensions": [
      { "type": "string", "page" },
      { "type": "string", "language" },
      { "type": "long", "name": "userId" }
    ]
  },
  "metricsSpec": [
    { "type": "count", "name": "count" },
    { "type": "doubleSum", "name": "bytes_added_sum", "fieldName": "bytes_added" },
    { "type": "doubleSum", "name": "bytes_deleted_sum", "fieldName": "bytes_deleted" }
  ],
  "granularitySpec": {
    "segmentGranularity": "day",
    "queryGranularity": "none",
    "intervals": [
      "2013-08-31/2013-09-01"
    ]
  }
}
```

##### `dataSource`
`dataSource` 位于 `dataSchema` -> `dataSource` 中，简单的标识了数据将被写入的数据源的名称，示例如下：
```
"dataSource": "my-first-datasource"
```
##### `timestampSpec`
`timestampSpec` 位于 `dataSchema` -> `timestampSpec` 中，用来配置 [主时间戳](#timestampspec), 示例如下：
```
"timestampSpec": {
  "column": "timestamp",
  "format": "auto"
}
```
> [!WARNING]
> 概念上，输入数据被读取后，Druid会以一个特定的顺序来对数据应用摄入规范： 首先 `flattenSpec`(如果有)，然后 `timestampSpec`, 然后 `transformSpec` ,最后是 `dimensionsSpec` 和 `metricsSpec`。在编写摄入规范时需要牢记这一点

`timestampSpec` 可以包含以下的部分：

<table>
    <thead>
        <th>字段</th>
        <th>描述</th>
        <th>默认值</th>
    </thead>
    <tbody>
        <tr>
            <td>column</td>
            <td>要从中读取主时间戳的输入行字段。<br><br>不管这个输入字段的名称是什么，主时间戳总是作为一个名为"__time"的列存储在您的Druid数据源中</td>
            <td>timestamp</td>
        </tr>
        <tr>
            <td>format</td>
            <td>
                时间戳格式，可选项有：
                <ul>
                    <li><code>iso</code>: 使用"T"分割的ISO8601，像"2000-01-01T01:02:03.456"</li>
                    <li><code>posix</code>: 自纪元以来的秒数</li>
                    <li><code>millis</code>: 自纪元以来的毫秒数</li>
                    <li><code>micro</code>: 自纪元以来的微秒数</li>
                    <li><code>nano</code>: 自纪元以来的纳秒数</li>
                    <li><code>auto</code>: 自动检测ISO或者毫秒格式</li>
                    <li>任何 <a href="http://joda-time.sourceforge.net/apidocs/org/joda/time/format/DateTimeFormat.html">Joda DateTimeFormat字符串</a></li>
                </ul>
            </td>
            <td>auto</td>
        </tr>
        <tr>
            <td>missingValue</td>
            <td>用于具有空或缺少时间戳列的输入记录的时间戳。应该是ISO8601格式，如<code>"2000-01-01T01:02:03.456"</code>。由于Druid需要一个主时间戳，因此此设置对于接收根本没有任何时间戳的数据集非常有用。</td>
            <td>none</td>
        </tr>
    </tbody>
</table>

##### `dimensionSpec`
`dimensionsSpec` 位于 `dataSchema` -> `dimensionsSpec`, 用来配置维度。示例如下：
```
"dimensionsSpec" : {
  "dimensions": [
    "page",
    "language",
    { "type": "long", "name": "userId" }
  ],
  "dimensionExclusions" : [],
  "spatialDimensions" : []
}
```

> [!WARNING]
> 概念上，输入数据被读取后，Druid会以一个特定的顺序来对数据应用摄入规范： 首先 `flattenSpec`(如果有)，然后 `timestampSpec`, 然后 `transformSpec` ,最后是 `dimensionsSpec` 和 `metricsSpec`。在编写摄入规范时需要牢记这一点

`dimensionsSpec` 可以包括以下部分：

| 字段 | 描述 | 默认值 |
|-|-|-|
| dimensions | 维度名称或者对象的列表，在 `dimensions` 和 `dimensionExclusions` 中不能包含相同的列。 <br><br> 如果该配置为一个空数组，Druid将会把所有未出现在 `dimensionExclusions` 中的非时间、非指标列当做字符串类型的维度列，参见[Inclusions and exclusions](#Inclusions-and-exclusions)。 | `[]` |
| dimensionExclusions | 在摄取中需要排除的列名称，在该配置中只支持名称，不支持对象。在 `dimensions` 和 `dimensionExclusions` 中不能包含相同的列。 | `[]` |
| spatialDimensions | 一个[空间维度](../Querying/spatialfilter.md)的数组 | `[]` |

###### `Dimension objects`
在 `dimensions` 列的每一个维度可以是一个名称，也可以是一个对象。 提供一个名称等价于提供了一个给定名称的 `string` 类型的维度对象。例如： `page` 等价于 `{"name": "page", "type": "string"}`。

维度对象可以有以下的部分：

| 字段 | 描述 | 默认值 |
|-|-|-|
| type | `string`, `long`, `float` 或者 `double` | `string` |
| name | 维度名称，将用作从输入记录中读取的字段名，以及存储在生成的段中的列名。<br><br> 注意： 如果想在摄取的时候重新命名列，可以使用 [`transformSpec`](#transformspec) | none（必填）|
| createBitmapIndex | 对于字符串类型的维度，是否应为生成的段中的列创建位图索引。创建位图索引需要更多存储空间，但会加快某些类型的筛选（特别是相等和前缀筛选）。仅支持字符串类型的维度。| `true` |

###### `Inclusions and exclusions`
##### `metricsSpec`
##### `granularitySpec`
##### `transformSpec`
##### 过时的 `dataSchema` 规范
#### `ioConfig`
#### `tuningConfig`