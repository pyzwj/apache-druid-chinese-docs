<!-- toc -->

## 集群部署

Apache Druid旨在作为可伸缩的容错集群进行部署。

在本文档中，我们将安装一个简单的集群，并讨论如何对其进行进一步配置以满足您的需求。

这个简单的集群将具有以下特点：
* 一个Master服务同时起Coordinator和Overlord进程
* 两个可伸缩、容错的Data服务来运行Historical和MiddleManager进程
* 一个Query服务，运行Druid Broker和Router进程

在生产中，我们建议根据您的特定容错需求部署多个Master服务器和多个Query服务器，但是您可以使用一台Master服务器和一台Query服务器将服务快速运行起来，然后再添加更多服务器。

### 选择硬件
#### 首次部署

如果您现在没有Druid集群，并打算首次以集群模式部署运行Druid，则本指南提供了一个包含预先配置的集群部署示例。

##### Master服务

Coordinator进程和Overlord进程负责处理集群的元数据和协调需求，它们可以运行在同一台服务器上。

在本示例中，我们将在等效于AWS[m5.2xlarge](https://aws.amazon.com/ec2/instance-types/m5/)实例的硬件环境上部署。

硬件规格为：

* 8核CPU
* 31GB内存

可以在`conf/druid/cluster/master`下找到适用于此硬件规格的Master示例服务配置。

##### Data服务

Historical和MiddleManager可以分配在同一台服务器上运行，以处理集群中的实际数据，这两个服务受益于CPU、内存和固态硬盘。

在本示例中，我们将在等效于AWS[i3.4xlarge](https://aws.amazon.com/cn/ec2/instance-types/i3/)实例的硬件环境上部署。

硬件规格为：
* 16核CPU
* 122GB内存
* 2 * 1.9TB 固态硬盘

可以在`conf/druid/cluster/data`下找到适用于此硬件规格的Data示例服务配置。

##### Query服务

Druid Broker服务接收查询请求，并将其转发到集群中的其他部分，同时其可以可选的配置内存缓存。 Broker服务受益于CPU和内存。

在本示例中，我们将在等效于AWS[m5.2xlarge](https://aws.amazon.com/ec2/instance-types/m5/)实例的硬件环境上部署。

硬件规格为：

* 8核CPU
* 31GB内存

您可以考虑将所有的其他开源UI工具或者查询依赖等与Broker服务部署在同一台服务器上。

可以在`conf/druid/cluster/query`下找到适用于此硬件规格的Query示例服务配置。

##### 其他硬件配置

上面的示例集群是从多种确定Druid集群大小的可能方式中选择的一个示例。

您可以根据自己的特定需求和限制选择较小/较大的硬件或较少/更多的服务器。

如果您的使用场景具有复杂的扩展要求，则还可以选择不将Druid服务混合部署（例如，独立的Historical Server）。

[基本集群调整指南]()中的信息可以帮助您进行决策，并可以调整配置大小。

#### 从单服务器环境迁移部署

如果您现在已有单服务器部署的环境，例如[单服务器部署示例](./chapter-3.md)中的部署，并且希望迁移到类似规模的集群部署，则以下部分包含一些选择Master/Data/Query服务等效硬件的准则。

##### Master服务

Master服务的主要考虑点是可用CPU以及用于Coordinator和Overlord进程的堆内存。

首先计算出来在单服务器环境下Coordinator和Overlord已分配堆内存之和，然后选择具有足够内存的Master服务硬件，同时还需要考虑到为服务器上其他进程预留一些额外的内存。

对于CPU，可以选择接近于单服务器环境核数1/4的硬件。

##### Data服务

在为集群Data服务选择硬件时，主要考虑可用的CPU和内存，可行时使用SSD存储。

在集群化部署时，出于容错的考虑，最好是部署多个Data服务。

在选择Data服务的硬件时，可以假定一个因子`N`，将原来的单服务器环境的CPU和内存除以`N`,然后在新集群中部署`N`个硬件规格缩小的Data服务。

##### Query服务

Query服务的硬件选择主要考虑可用的CPU、Broker服务的堆内和堆外内存、Router服务的堆内存。

首先计算出来在单服务器环境下Broker和Router已分配堆内存之和，然后选择可以覆盖Broker和Router内存的Query服务硬件，同时还需要考虑到为服务器上其他进程预留一些额外的内存。

对于CPU，可以选择接近于单服务器环境核数1/4的硬件。

[基本集群调优指南]()包含有关如何计算Broker和Router服务内存使用量的信息。

### 选择操作系统

我们建议运行您喜欢的Linux发行版，同时还需要：

* **Java 8**

> [!WARNING]
> Druid服务运行依赖Java 8，可以使用环境变量`DRUID_JAVA_HOME`或`JAVA_HOME`指定在何处查找Java,有关更多详细信息，请运行`verify-java`脚本。

### 下载发行版

首先，下载并解压缩发布安装包。最好首先在单台计算机上执行此操作，因为您将编辑配置，然后将修改后的配置分发到所有服务器上。

[下载](https://www.apache.org/dyn/closer.cgi?path=/druid/0.17.0/apache-druid-0.17.0-bin.tar.gz)Druid最新0.17.0release安装包

在终端中运行以下命令来提取Druid

```
tar -xzf apache-druid-0.17.0-bin.tar.gz
cd apache-druid-0.17.0
```

在安装包中有以下文件：

* `LICENSE`和`NOTICE`文件
* `bin/*` - 启停等脚本
* `conf/druid/cluster/*` - 用于集群部署的模板配置
* `extensions/*` - Druid核心扩展
* `hadoop-dependencies/*` - Druid Hadoop依赖
* `lib/*` - Druid核心库和依赖
* `quickstart/*` - 与[快速入门](./chapter-2.md)相关的文件

我们主要是编辑`conf/druid/cluster/`中的文件。

#### 从单服务器环境迁移部署

在以下各节中，我们将在`conf/druid/cluster`下编辑配置。

如果您已经有一个单服务器部署，请将您的现有配置复制到`conf/druid /cluster`以保留您所做的所有配置更改。

### 配置元数据存储和深度存储
#### 从单服务器环境迁移部署

如果您已经有一个单服务器部署，并且希望在整个迁移过程中保留数据，请在更新元数据/深层存储配置之前，按照[元数据迁移]()和[深层存储迁移]()中的说明进行操作。

这些指南针对使用Derby元数据存储和本地深度存储的单服务器部署。 如果您已经在单服务器集群中使用了非Derby元数据存储，则可以在新集群中可以继续使用当前的元数据存储。

这些指南还提供了有关从本地深度存储迁移段的信息。集群部署需要分布式深度存储，例如S3或HDFS。 如果单服务器部署已在使用分布式深度存储，则可以在新集群中继续使用当前的深度存储。

#### 元数据存储

在`conf/druid/cluster/_common/common.runtime.properties`中，使用您将用作元数据存储的服务器地址来替换"metadata.storage.*":

* `druid.metadata.storage.connector.connectURI`
* `druid.metadata.storage.connector.host`

在生产部署中，我们建议运行专用的元数据存储，例如具有复制功能的MySQL或PostgreSQL，与Druid服务器分开部署。

[MySQL扩展]()和[PostgreSQL]()扩展文档包含有关扩展配置和初始数据库安装的说明。

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


