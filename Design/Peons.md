<!-- toc -->
## Peons
### 配置
对于Apache Druid Peon配置，可以参见 [Peon查询配置](../Configuration/configuration.md) 和 [额外的Peon配置](../Configuration/configuration.md)

### HTTP
对于Peon的API接口，详见 [Peon API](../Operations/api.md#Peon)

Peon在单个JVM中运行单个任务。MiddleManager负责创建运行任务的Peon。Peon应该很少（如果为了测试目的）自己运行。

### 运行
Peon应该很少独立于MiddleManager，除非出于开发目的。

```
org.apache.druid.cli.Main internal peon <task_file> <status_file>
```

任务文件包含任务JSON对象。状态文件指示将输出任务状态的位置。