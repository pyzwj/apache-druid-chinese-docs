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


#### KafkaSupervisorIOConfig
### 操作
#### 获取supervisor的状态报告
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