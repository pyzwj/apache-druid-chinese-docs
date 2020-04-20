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