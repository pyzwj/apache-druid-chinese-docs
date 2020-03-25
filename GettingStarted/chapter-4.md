## 集群部署

Apache Druid旨在作为可伸缩的容错集群进行部署。

在本文档中，我们将安装一个简单的集群，并讨论如何对其进行进一步配置以满足您的需求。

这个简单的集群将具有以下特点：
* 一个Master服务同时起Coordinator和Overlord进程
* 两个可伸缩、容错的Data服务来运行Historical和MiddleManager进程
* 一个Query服务，运行Druid Broker和Router进程

在生产中，我们建议根据您的特定容错需求部署多个Master服务器和多个Query服务器，但是您可以使用一台Master服务器和一台Query服务器将服务快速运行起来，然后再添加更多服务器。

### 选择硬件
#### 重新部署
##### Master服务
##### Data服务
##### Query服务
##### 其他硬件配置
#### 从单服务器环境迁移部署
##### Master服务
##### Data服务
##### Query服务
### 选择操作系统
### 下载安装包
#### 从单服务器环境迁移部署
### 配置元数据存储和深度存储
#### 从单服务器环境迁移部署
#### 元数据存储
#### 深度存储
##### S3
##### HDFS
### Hadoop连接配置
### Zookeeper连接配置
### 配置调整
#### 从单服务器环境迁移部署
##### Master服务
##### Data服务
##### Query服务
#### 重新部署
### 开启端口(如果使用了防火墙)
#### Master服务
#### Data服务
#### Query服务
### 启动Master服务
#### 不带Zookeeper启动
#### 带Zookeeper启动
### 启动Data服务
### 启动Query服务
### 加载数据


