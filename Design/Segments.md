<!-- toc -->
## 段
ApacheDruid将索引存储在按时间分区的*段文件*中。在基本设置中，通常为每个时间间隔创建一个段文件，其中时间间隔可在 `granularitySpec` 的`segmentGranularity` 参数中配置。为了使Druid在繁重的查询负载下运行良好，段文件大小必须在建议的300MB-700MB范围内。如果段文件大于此范围，请考虑更改时间间隔的粒度，或者对数据进行分区，并在 `partitionsSpec` 中调整 `targetPartitionSize`（此参数的建议起点是500万行）。有关更多信息，请参阅下面的**分片部分**和[批处理摄取]()文档的**分区规范**部分。

### 段文件的核心数据结构

在这里，我们描述段文件的内部结构，它本质上是*列式*的：每列的数据在单独的数据结构中。通过分别存储每一列，Druid可以通过只扫描查询实际需要的列来减少查询延迟。有三种基本列类型：**时间戳列、维度列和指标列**，如下图所示：

![](img/druid-column-types.png)

timestamp和metric列很简单：在底层，每个列都是用LZ4压缩的整数或浮点值数组。一旦查询知道需要选择哪些行，它只需解压缩这些行，提取相关行，然后使用所需的聚合运算符进行计算。与所有列一样，如果不查询一个列，则跳过该列的数据。

dimension列是不同的，因为它们支持过滤和聚合操作，所以每一个维度都需要以下三种数据结构：

1. 一个将值（通常被当做字符串）映射到整数id的字典
2. 一个使用第一步的字典进行编码的列值的列表
3. 对于列中每一个不同的值，标识哪些行包含该值的位图

为什么需要这三种数据结构？字典简单地将字符串值映射到整数id，以便于在（2）和（3）中可以紧凑的表示。（3）中的位图（也称*倒排索引*）可以进行快速过滤操作（特别是，位图便于快速进行AND和OR操作）。 最后，`GroupBy` 和 `TopN` 查询需要（2）中的值列表。换句话说，仅基于过滤器的聚合指标是不需要（2）中存储的维度值列表的。

要具体了解这些数据结构，请考虑上面示例数据中的"page"列,表示此维度的三个数据结构如下图所示：

```
1: Dictionary that encodes column values
  {
    "Justin Bieber": 0,
    "Ke$ha":         1
  }

2: Column data
  [0,
   0,
   1,
   1]

3: Bitmaps - one for each unique value of the column
  value="Justin Bieber": [1,1,0,0]
  value="Ke$ha":         [0,0,1,1]
```

注意，位图与前两个数据结构不同：前两个数据结构在数据大小上呈线性增长（在最坏的情况下），而位图部分的大小是数据大小 * 列基数的乘积。不过，压缩在这里会有帮助，因为我们知道对于"列数据"中的每一行，只有一个具有非零项的位图,这意味着高基数列将具有非常稀疏的、高度可压缩的位图。Druid使用特别适合位图的压缩算法（如Roaring位图压缩）来利用这一点特性。

**多值列**

如果数据源使用多值列，那么段文件中的数据结构看起来有点不同。让我们假设在上面的例子中，第二行同时标记了"Ke$ha"和"Justin Bieber"主题。在这种情况下，这三种数据结构现在看起来如下：

```
1: Dictionary that encodes column values
  {
    "Justin Bieber": 0,
    "Ke$ha":         1
  }

2: Column data
  [0,
   [0,1],  <--Row value of multi-value column can have array of values
   1,
   1]

3: Bitmaps - one for each unique value
  value="Justin Bieber": [1,1,0,0]
  value="Ke$ha":         [0,1,1,1]
                            ^
                            |
                            |
    Multi-value column has multiple non-zero entries
```

请注意列数据和Ke$ha位图中第二行的更改。如果一行中的某一列有多个值，则其在“列数据”中的条目是一个数组。此外，在“列数据”中有n个值的行在位图中将有n个非零值项。