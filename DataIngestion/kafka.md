<!-- toc -->
## Apache Kafka 摄取数据

Kafka索引服务支持在Overlord上配置*supervisors*，supervisors通过管理Kafka索引任务的创建和生存期来便于从Kafka摄取数据。这些索引任务使用Kafka自己的分区和偏移机制读取事件，因此能够保证只接收一次（**exactly-once**）。supervisor监视索引任务的状态，以便于协调切换、管理故障，并确保维护可伸缩性和复制要求。

这个服务由 `druid-kafka-indexing-service` 这个druid核心扩展（详情请见 [扩展列表](../Development/extensions.md）所提供。

> [!WARNING]
> Kafka索引服务支持在Kafka 0.11.x中引入的事务主题。这些更改使Druid使用的Kafka消费者与旧的brokers不兼容。在使用此功能之前，请确保您的Kafka broker版本为0.11.x或更高版本。如果您使用的是旧版本的Kafka brokers，请参阅《[Kafka升级指南](https://kafka.apache.org/documentation/#upgrade)》。

### 教程

本页包含基于Apache Kafka的摄取的参考文档。同样，您可以查看 [Apache Kafka教程](../Tutorials/chapter-2.md) 中的加载。

### 提交一个supervisor规范

Kafka索引服务需要同时在Overlord和MiddleManagers中加载 `druid-kafka-indexing-service` 扩展。 用于一个数据源的supervisor通过向 `http://<OVERLORD_IP>:<OVERLORD_PORT>/druid/indexer/v1/supervisor` 发送一个HTTP POST请求来启动，例如：

```
curl -X POST -H 'Content-Type: application/json' -d @supervisor-spec.json http://localhost:8090/druid/indexer/v1/supervisor
```

一个示例supervisor规范如下：
```
{
  "type": "kafka",
  "dataSchema": {
    "dataSource": "metrics-kafka",
    "timestampSpec": {
      "column": "timestamp",
      "format": "auto"
    },
    "dimensionsSpec": {
      "dimensions": [],
      "dimensionExclusions": [
        "timestamp",
        "value"
      ]
    },
    "metricsSpec": [
      {
        "name": "count",
        "type": "count"
      },
      {
        "name": "value_sum",
        "fieldName": "value",
        "type": "doubleSum"
      },
      {
        "name": "value_min",
        "fieldName": "value",
        "type": "doubleMin"
      },
      {
        "name": "value_max",
        "fieldName": "value",
        "type": "doubleMax"
      }
    ],
    "granularitySpec": {
      "type": "uniform",
      "segmentGranularity": "HOUR",
      "queryGranularity": "NONE"
    }
  },
  "tuningConfig": {
    "type": "kafka",
    "maxRowsPerSegment": 5000000
  },
  "ioConfig": {
    "topic": "metrics",
    "inputFormat": {
      "type": "json"
    },
    "consumerProperties": {
      "bootstrap.servers": "localhost:9092"
    },
    "taskCount": 1,
    "replicas": 1,
    "taskDuration": "PT1H"
  }
}
```

### supervisor配置

| 字段 | 描述 | 是否必须 |
|-|-|-|-|
| `type` | supervisor类型， 总是 `kafka` | 是 |
| `dataSchema` | Kafka索引服务在摄取时使用的schema。详情见 [dataSchema](ingestion.md#dataschema)  | 是 |
| `ioConfig` | 用于配置supervisor和索引任务的KafkaSupervisorIOConfig，详情见以下 | 是 |
| `tuningConfig` | 用于配置supervisor和索引任务的KafkaSupervisorTuningConfig，详情见以下 | 是 |

#### KafkaSupervisorTuningConfig

`tuningConfig` 是可选的， 如果未被配置的话，则使用默认的参数。

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| `type` | String | 索引任务类型， 总是 `kafka` | 是 |
| `maxRowsInMemory` | Integer | 在持久化之前在内存中聚合的最大行数。该数值为聚合之后的行数，所以它不等于原始输入事件的行数，而是事件被聚合后的行数。 通常用来管理所需的JVM堆内存。 使用 `maxRowsInMemory * (2 + maxPendingPersists) ` 来当做索引任务的最大堆内存。通常用户不需要设置这个值，但是也需要根据数据的特点来决定，如果行的字节数较短，用户可能不想在内存中存储一百万行，应该设置这个值 | 否（默认为 1000000）|
| `maxBytesInMemory` | Long | 在持久化之前在内存中聚合的最大字节数。这是基于对内存使用量的粗略估计，而不是实际使用量。通常这是在内部计算的，用户不需要设置它。 索引任务的最大内存使用量是 `maxRowsInMemory * (2 + maxPendingPersists) ` | 否（默认为最大JVM内存的 1/6） |
| `maxRowsPerSegment` | Integer | 聚合到一个段中的行数，该数值为聚合后的数值。 当 `maxRowsPerSegment` 或者 `maxTotalRows` 有一个值命中的时候，则触发handoff（数据落盘后传到深度存储）， 该动作也会按照每 `intermediateHandoffPeriod` 时间间隔发生一次。 | 否（默认为5000000） |
| `maxTotalRows` | Long | 所有段的聚合后的行数，该值为聚合后的行数。当 `maxRowsPerSegment` 或者 `maxTotalRows` 有一个值命中的时候，则触发handoff（数据落盘后传到深度存储）， 该动作也会按照每 `intermediateHandoffPeriod` 时间间隔发生一次。 | 否（默认为unlimited）|
| `intermediateHandoffPeriod` | ISO8601 Period | 确定触发持续化存储的周期 | 否（默认为 PT10M）|
| `maxPendingPersists` | Integer | 正在等待但启动的持久化过程的最大数量。 如果新的持久化任务超过了此限制，则在当前运行的持久化完成之前，摄取将被阻止。索引任务的最大内存使用量是 `maxRowsInMemory * (2 + maxPendingPersists) ` | 否（默认为0，意味着一个持久化可以与摄取同时运行，而没有一个可以排队）|
| `indexSpec` | Object | 调整数据被如何索引。详情可以见 [indexSpec](#indexspec) | 否 |
| `indexSpecForIntermediatePersists` | | 定义要在索引时用于中间持久化临时段的段存储格式选项。这可用于禁用中间段上的维度/度量压缩，以减少最终合并所需的内存。但是，在中间段上禁用压缩可能会增加页缓存的使用，而在它们被合并到发布的最终段之前使用它们，有关可能的值，请参阅IndexSpec。 | 否（默认与 `indexSpec` 相同） |
| `reportParseExceptions` | Boolean | *已废弃*。如果为true，则在解析期间遇到的异常即停止摄取；如果为false，则将跳过不可解析的行和字段。将 `reportParseExceptions` 设置为 `true` 将覆盖`maxParseExceptions` 和 `maxSavedParseExceptions` 的现有配置，将`maxParseExceptions` 设置为 `0` 并将 `maxSavedParseExceptions` 限制为不超过1。 | 否（默认为false）|
| `handoffConditionTimeout` | Long | 段切换（持久化）可以等待的毫秒数（超时时间）。 该值要被设置为大于0的数，设置为0意味着将会一直等待不超时 | 否（默认为0）|
| `resetOffsetAutomatically` | Boolean | 控制当Druid需要读取Kafka中不可用的消息时的行为，比如当发生了 `OffsetOutOfRangeException` 异常时。 <br> 如果为false，则异常将抛出，这将导致任务失败并停止接收。如果发生这种情况，则需要手动干预来纠正这种情况；可能使用 [重置 Supervisor API](../Operations/api.md#Supervisor)。此模式对于生产非常有用，因为它将使您意识到摄取的问题。 <br> 如果为true，Druid将根据 `useEarliestOffset` 属性的值（`true` 为 `earliest`，`false` 为 `latest`）自动重置为Kafka中可用的较早或最新偏移量。请注意，这可能导致数据在您不知情的情况下*被丢弃*（如果`useEarliestOffset` 为 `false`）或 *重复*（如果 `useEarliestOffset` 为 `true`）。消息将被记录下来，以标识已发生重置，但摄取将继续。这种模式对于非生产环境非常有用，因为它将使Druid尝试自动从问题中恢复，即使这些问题会导致数据被安静删除或重复。 <br> 该特性与Kafka的 `auto.offset.reset` 消费者属性很相似 | 否（默认为false）|
| `workerThreads` | Integer | supervisor用于异步操作的线程数。| 否（默认为: min(10, taskCount)） |
| `chatThreads` | Integer | 与索引任务的会话线程数 | 否（默认为：min(10, taskCount * replicas)）|
| `chatRetries` | Integer | 在任务没有响应之前，将重试对索引任务的HTTP请求的次数 | 否（默认为8）|
| `httpTimeout` | ISO8601 Period | 索引任务的HTTP响应超时 | 否（默认为PT10S）|
| `shutdownTimeout` | ISO8601 Period | supervisor尝试优雅的停掉一个任务的超时时间 | 否（默认为：PT80S）|
| `offsetFetchPeriod` | ISO8601 Period | supervisor查询Kafka和索引任务以获取当前偏移和计算滞后的频率 | 否（默认为PT30S，最小为PT5S）|
| `segmentWriteOutMediumFactory` | Object | 创建段时要使用的段写入介质。更多信息见下文。| 否（默认不指定，使用来源于 `druid.peon.defaultSegmentWriteOutMediumFactory.type` 的值）|
| `intermediateHandoffPeriod` | ISO8601 Period | 段发生切换的频率。当 `maxRowsPerSegment` 或者 `maxTotalRows` 有一个值命中的时候，则触发handoff（数据落盘后传到深度存储）， 该动作也会按照每 `intermediateHandoffPeriod` 时间间隔发生一次。 | 否（默认为：P2147483647D）|
| `logParseExceptions` | Boolean | 如果为true，则在发生解析异常时记录错误消息，其中包含有关发生错误的行的信息。| 否（默认为false）|
| `maxParseExceptions` | Integer | 任务停止接收之前可发生的最大分析异常数。如果设置了 `reportParseExceptions`，则该值会被重写。| 否（默认为unlimited）|
| `maxSavedParseExceptions` | Integer | 当出现解析异常时，Druid可以跟踪最新的解析异常。"maxSavedParseExceptions"决定将保存多少个异常实例。这些保存的异常将在 [任务完成报告](taskrefer.md#任务报告) 中的任务完成后可用。如果设置了`reportParseExceptions`，则该值会被重写。 | 否（默认为0）|

##### IndexSpec

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|-|
| `bitmap` | Object | 位图索引的压缩格式。 应该是一个JSON对象，详情见以下 | 否（默认为 `roaring`）|
| `dimensionCompression` | String | 维度列的压缩格式。 从 `LZ4`, `LZF` 或者 `uncompressed` 选择 | 否（默认为 `LZ4`）|
| `metricCompression` | String | Metrics列的压缩格式。 从 `LZ4`, `LZF`, `uncompressed` 或者 `none` 选择 | 否（默认为 `LZ4`）|
| `longEncoding` | String | 类型为long的Metric列和维度列的编码格式。从 `auto` 或者 `longs` 中选择。`auto`编码是根据列基数使用偏移量或查找表对值进行编码，并以可变大小存储它们。`longs` 按原样存储值，每个值8字节。 | 否（默认为 `longs`）|

**Bitmap类型**

对于Roaring位图：

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| `type` | String | 必须为 `roaring` | 是 |
| `compressRunOnSerialization` | Boolean | 使用一个运行长度编码，可以更节省空间 | 否（默认为 `true` ）|

对于Concise位图：

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| `type` | String | 必须为 `concise` | 是 |

##### SegmentWriteOutMediumFactory

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| `type` | String | 对于可用选项，可以见 [额外的Peon配置：SegmentWriteOutMediumFactory](../Configuration/configuration.md#SegmentWriteOutMediumFactory) | 是 | 

#### KafkaSupervisorIOConfig

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| `topic` | String | 要读取数据的Kafka主题。这必须是一个特定的主题，因为不支持主题模式 | 是 |
| `inputFormat` | Object | [`inputFormat`](dataformats.md#inputformat) 指定如何解析输入数据。 看 [下边部分](#指定输入数据格式) 查看指定输入格式的详细信息。 | 是 |
| `consumerProperties` | Map<String, Object> | 传给Kafka消费者的一组属性map。必须得包含 `bootstrap.servers` 的属性，其值为Kafka Broker列表，格式为: `<BROKER_1>:<PORT_1>,<BROKER_2>:<PORT_2>,...`。 对于SSL连接，`keystore`, `truststore` 和 `key` 密码可以被以一个字符串密码或者 [密码Provider](../Operations/passwordproviders.md) 来提供 | 是 |
| `pollTimeout` | Long | Kafka消费者拉取消息记录的超时等待时间，毫秒单位 | 否（默认为100）|
| `replicas` | Integer | 副本的数量，1意味着一个单一任务（无副本）。副本任务将始终分配给不同的worker，以提供针对流程故障的恢复能力。| 否（默认为1）|
| `taskCount` | Integer | *一个副本集* 中*读取*任务的最大数量。 这意味着读取任务的最大的数量将是 `taskCount * replicas`, 任务总数（*读取 + 发布*）是大于这个数字的。 详情可以看下边的 [容量规划](#容量规划)。 如果 `taskCount > {numKafkaPartitions}`, 读取任务的数量会小于 `taskCount` | 否（默认为1）|
| `taskDuration` | ISO8601 Period | 任务停止读取数据、开始发布段之前的时间长度 | 否（默认为PT1H）|
| `startDelay` | ISO8601 Period | supervisor开始管理任务之前的等待时间 | 否（默认为PT5S）|
| `useEarliestOffset` | Boolean | 如果supervisor是第一次管理数据源，它将从Kafka获得一组起始偏移。此标志确定它是检索Kafka中的最早偏移量还是最新偏移量。在正常情况下，后续任务将从先前段结束的位置开始，因此此标志将仅在首次运行时使用。 | 否（默认false）|
| `completionTimeout` | ISO8601 Period | 声明发布任务为失败并终止它 之前等待的时间长度。如果设置得太低，则任务可能永远不会发布。任务的发布时刻大约在 `taskDuration` (任务持续)时间过后开始。 | 否（默认为PT30M）|
| `lateMessageRejectionStartDateTime` | ISO8601 DateTime | 用来配置一个时间，当消息时间戳早于此日期时间的时候，消息被拒绝。 例如，如果该值设置为 `2016-01-01T11:00Z`, supervisor在 *`2016-01-01T12:00Z`* 创建了一个任务，时间戳早于 *2016-01-01T11:00Z* 的消息将会被丢弃。如果您的数据流有延迟消息，并且您有多个需要在同一段上操作的管道（例如实时和夜间批处理摄取管道），这可能有助于防止并发问题。 | 否（默认为none）|
| `lateMessageRejectionPeriod` | ISO8601 Period | 用来配置一个时间周期，当消息时间戳早于此周期的时候，消息被拒绝。例如，如果该值设置为 `PT1H`, supervisor 在 `2016-01-01T12:00Z` 创建了一个任务，则时间戳早于 `2016-01-01T11:00Z` 的消息将被丢弃。 如果您的数据流有延迟消息，并且您有多个需要在同一段上操作的管道（例如实时和夜间批处理摄取管道），这可能有助于防止并发问题。 **请特别注意**，`lateMessageRejectionPeriod` 和 `lateMessageRejectionStartDateTime` 仅一个可以被指定。 | 否（默认none）|
| `earlyMessageRejectionPeriod` | ISO8601 Period | 用来配置一个时间周期，当消息时间戳晚于此周期的时候，消息被拒绝。 例如，如果该值设置为 `PT1H`,supervisor 在 `2016-01-01T12:00Z` 创建了一个任务，则时间戳晚于 `2016-01-01T14:00Z` 的消息将被丢弃。**注意**，任务有时会超过其任务持续时间，例如，在supervisor故障转移的情况下。如果将 `earlyMessageRejectionPeriod` 设置得太低，则每当任务运行超过其最初配置的任务持续时间时，可能会导致消息意外丢弃。| 否（默认none）|

##### 指定输入数据格式

Kafka索引服务同时支持通过 [`inputFormat`](dataformats.md#inputformat) 和 [`parser`](dataformats.md#parser) 来指定数据格式。 `inputFormat` 是一种新的且推荐的用于Kafka索引服务中指定数据格式的方式，但是很遗憾的是目前它还不支持过时的 `parser` 所有支持的所有格式（未来会支持）。

`inputFormat` 支持的格式包括 [`csv`](dataformats.md#csv), [`delimited`](dataformats.md#TSV(Delimited)), [`json`](dataformats.md#json)。可以使用 `parser` 来读取 [`avro_stream`](dataformats.md#AvroStreamParser), [`protobuf`](dataformats.md#ProtobufParser), [`thrift`](../Development/thrift.md) 格式的数据。

### 操作

本节描述了一些supervisor API如何在Kafka索引服务中具体工作。对于所有的supervisor API，请查看 [Supervisor APIs](../Operations/api.md#Supervisor)

#### 获取supervisor的状态报告

`GET /druid/indexer/v1/supervisor/<supervisorId>/status` 返回由给定supervisor管理的任务当前状态的快照报告。报告中包括Kafka报告的最新偏移量、每个分区的使用者延迟，以及所有分区的聚合延迟。如果supervisor没有收到来自Kafka的最新偏移响应，则每个分区的使用者延迟可以报告为负值。聚合滞后值将始终大于等于0。

状态报告还包含supervisor的状态和最近引发的异常列表（报告为`recentErrors`，其最大大小可以使用 `druid.supervisor.maxStoredExceptionEvents` 配置进行控制）。有两个字段与supervisor的状态相关- `state` 和 `detailedState`。`state` 字段将始终是少数适用于任何类型的supervisor的通用状态之一，而 `detailedState` 字段将包含一个更具描述性的、特定实现的状态，该状态可以比通用状态字段更深入地了解supervisor的活动。

`state` 可能的值列表为：[`PENDING`, `RUNNING`, `SUSPENDED`, `STOPPING`, `UNHEALTHY_SUPERVISOR`, `UNHEALTHY_TASKS`]

`detailedState`值与它们相应的 `state` 映射关系如下：

| Detailed State | 相应的State | 描述 |
|-|-|-|
| UNHEALTHY_SUPERVISOR | UNHEALTHY_SUPERVISOR | supervisor在过去的 `druid.supervisor.unhealthinessThreshold` 内已经发生了错误 |
| UNHEALTHY_TASKS | UNHEALTHY_TASKS | 过去 `druid.supervisor.taskUnhealthinessThreshold` 内的任务全部失败了 |
| UNABLE_TO_CONNECT_TO_STREAM | UNHEALTHY_SUPERVISOR | supervisor遇到与Kafka的连接问题，过去没有成功连接过 |
| LOST_CONTACT_WITH_STREAM | UNHEALTHY_SUPERVISOR | supervisor遇到与Kafka的连接问题，但是在过去成功连接过 |
| PENDING（仅在第一次迭代中）| PENDING | supervisor已初始化，尚未开始连接到流 |
| CONNECTING_TO_STREAM（仅在第一次迭代中） | RUNNING | supervisor正在尝试连接到流并更新分区数据 |
| DISCOVERING_INITIAL_TASKS（仅在第一次迭代中） | RUNNING | supervisor正在发现已在运行的任务 |
| CREATING_TASKS（仅在第一次迭代中） | RUNNING | supervisor正在创建任务并发现状态 |
| RUNNING | RUNNING | supervisor已启动任务，正在等待任务持续时间结束 |
| SUSPENDED | SUSPENDED | supervisor被挂起 |
| STOPPING | STOPPING | supervisor正在停止 |

在supervisor运行循环的每次迭代中，supervisor按顺序完成以下任务：

1. 从Kafka获取分区列表并确定每个分区的起始偏移量（如果继续，则基于最后处理的偏移量，如果这是一个新主题，则从流的开始或结束开始）。
2. 发现正在写入supervisor数据源的任何正在运行的索引任务，如果这些任务与supervisor的配置匹配，则采用这些任务，否则发出停止的信号。
3. 向每个受监视的任务发送状态请求，以更新我们对受监视任务的状态的视图。
4. 处理已超过 `taskDuration(任务持续时间)` 且应从读取状态转换为发布状态的任务。
5. 处理已完成发布的任务，并发出停止冗余副本任务的信号。
6. 处理失败的任务并清理supervisor的内部状态。
7. 将正常任务列表与请求的 `taskCount` 和 `replicas` 进行比较，并根据需要创建其他任务。


#### 获取supervisor摄取状态报告
#### supervisor健康检测
#### 更新现有的supervisor
#### 暂停和恢复supervisors
#### 重置supervisors
#### 终止supervisors
#### 容量规划
#### supervisor持久化
#### schema/配置变更
#### 部署注意