<!-- toc -->

### 快速开始

在本快速入门教程中，我们将下载Druid并将其安装在一台服务器上，完成初始安装后，向集群中加载数据。

在开始快速入门之前，阅读[Druid概述](./chapter-1.md)和[数据摄取概述](../DataIngestion/index.md)会很有帮助，因为当前教程会引用这些页面上讨论的概念。

#### 预备条件
##### 软件
* **Java 8(8u92+)**
* Linux, Mac OS X, 或者其他类UNIX系统（Windows不支持）

> [!WARNING]
> Druid服务运行依赖Java 8，可以使用环境变量`DRUID_JAVA_HOME`或`JAVA_HOME`指定在何处查找Java,有关更多详细信息，请运行`verify-java`脚本。

##### 硬件

Druid安装包提供了几个[单服务器配置](./chapter-3.md)的示例，以及使用这些配置启动Druid进程的脚本。

如果您正在使用便携式等小型计算机上运行服务，则配置为4CPU/16GB RAM环境的`micro-quickstart`配置是一个不错的选择。

如果您打算在本教程之外使用单机部署进行进一步试验评估，则建议使用比`micro-quickstart`更大的配置。

#### 入门开始

[下载](https://www.apache.org/dyn/closer.cgi?path=/druid/0.17.0/apache-druid-0.17.0-bin.tar.gz)Druid最新0.17.0release安装包

在终端中运行以下命令来提取Druid

```
tar -xzf apache-druid-0.17.0-bin.tar.gz
cd apache-druid-0.17.0
```

在安装包中有以下文件：

* `LICENSE`和`NOTICE`文件
* `bin/*` - 启停等脚本
* `conf/*` - 用于单节点部署和集群部署的示例配置
* `extensions/*` - Druid核心扩展
* `hadoop-dependencies/*` - Druid Hadoop依赖
* `lib/*` - Druid核心库和依赖
* `quickstart/*` - 配置文件，样例数据，以及快速入门教材的其他文件

#### 启动服务

以下命令假定您使用的是`micro-quickstart`单机配置，如果使用的是其他配置，在`bin`目录下有每一种配置对应的脚本，如`bin/start-single-server-small`

在`apache-druid-0.17.0`安装包的根目录下执行命令：

```
./bin/start-micro-quickstart
```
然后将在本地计算机上启动Zookeeper和Druid服务实例，例如：

```
$ ./bin/start-micro-quickstart
[Fri May  3 11:40:50 2019] Running command[zk], logging to[/apache-druid-0.17.0/var/sv/zk.log]: bin/run-zk conf
[Fri May  3 11:40:50 2019] Running command[coordinator-overlord], logging to[/apache-druid-0.17.0/var/sv/coordinator-overlord.log]: bin/run-druid coordinator-overlord conf/druid/single-server/micro-quickstart
[Fri May  3 11:40:50 2019] Running command[broker], logging to[/apache-druid-0.17.0/var/sv/broker.log]: bin/run-druid broker conf/druid/single-server/micro-quickstart
[Fri May  3 11:40:50 2019] Running command[router], logging to[/apache-druid-0.17.0/var/sv/router.log]: bin/run-druid router conf/druid/single-server/micro-quickstart
[Fri May  3 11:40:50 2019] Running command[historical], logging to[/apache-druid-0.17.0/var/sv/historical.log]: bin/run-druid historical conf/druid/single-server/micro-quickstart
[Fri May  3 11:40:50 2019] Running command[middleManager], logging to[/apache-druid-0.17.0/var/sv/middleManager.log]: bin/run-druid middleManager conf/druid/single-server/micro-quickstart
```

所有的状态（例如集群元数据存储和服务的segment文件）将保留在`apache-druid-0.17.0`软件包根目录下的`var`目录中, 服务的日志位于 `var/sv`。

稍后，如果您想停止服务，请按`CTRL-C`退出`bin/start-micro-quickstart`脚本，该脚本将终止Druid进程。

集群启动后，可以访问[http://localhost:8888](http://localhost:8888)来Druid控制台，控制台由Druid Router进程启动。

![tutorial-quickstart](img/tutorial-quickstart-01.png)

所有Druid进程完全启动需要花费几秒钟。 如果在启动服务后立即打开控制台，则可能会看到一些可以安全忽略的错误。

#### 加载数据
##### 教程使用的数据集
##### 数据加载
##### 重置集群状态