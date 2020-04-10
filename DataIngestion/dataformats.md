<!-- toc -->
## 数据格式
Apache Druid可以接收JSON、CSV或TSV等分隔格式或任何自定义格式的非规范化数据。尽管文档中的大多数示例使用JSON格式的数据，但将Druid配置为接收任何其他分隔数据并不困难。我们欢迎对新格式的任何贡献。

此页列出了Druid支持的所有默认和核心扩展数据格式。有关社区扩展支持的其他数据格式，请参阅我们的 [社区扩展列表](../Configuration/extensions.md#社区扩展)。

### 格式化数据

下面的示例显示了在Druid中原生支持的数据格式：

*JSON*
```
{"timestamp": "2013-08-31T01:02:33Z", "page": "Gypsy Danger", "language" : "en", "user" : "nuclear", "unpatrolled" : "true", "newPage" : "true", "robot": "false", "anonymous": "false", "namespace":"article", "continent":"North America", "country":"United States", "region":"Bay Area", "city":"San Francisco", "added": 57, "deleted": 200, "delta": -143}
{"timestamp": "2013-08-31T03:32:45Z", "page": "Striker Eureka", "language" : "en", "user" : "speed", "unpatrolled" : "false", "newPage" : "true", "robot": "true", "anonymous": "false", "namespace":"wikipedia", "continent":"Australia", "country":"Australia", "region":"Cantebury", "city":"Syndey", "added": 459, "deleted": 129, "delta": 330}
{"timestamp": "2013-08-31T07:11:21Z", "page": "Cherno Alpha", "language" : "ru", "user" : "masterYi", "unpatrolled" : "false", "newPage" : "true", "robot": "true", "anonymous": "false", "namespace":"article", "continent":"Asia", "country":"Russia", "region":"Oblast", "city":"Moscow", "added": 123, "deleted": 12, "delta": 111}
{"timestamp": "2013-08-31T11:58:39Z", "page": "Crimson Typhoon", "language" : "zh", "user" : "triplets", "unpatrolled" : "true", "newPage" : "false", "robot": "true", "anonymous": "false", "namespace":"wikipedia", "continent":"Asia", "country":"China", "region":"Shanxi", "city":"Taiyuan", "added": 905, "deleted": 5, "delta": 900}
{"timestamp": "2013-08-31T12:41:27Z", "page": "Coyote Tango", "language" : "ja", "user" : "cancer", "unpatrolled" : "true", "newPage" : "false", "robot": "true", "anonymous": "false", "namespace":"wikipedia", "continent":"Asia", "country":"Japan", "region":"Kanto", "city":"Tokyo", "added": 1, "deleted": 10, "delta": -9}
```

*CSV*
```
2013-08-31T01:02:33Z,"Gypsy Danger","en","nuclear","true","true","false","false","article","North America","United States","Bay Area","San Francisco",57,200,-143
2013-08-31T03:32:45Z,"Striker Eureka","en","speed","false","true","true","false","wikipedia","Australia","Australia","Cantebury","Syndey",459,129,330
2013-08-31T07:11:21Z,"Cherno Alpha","ru","masterYi","false","true","true","false","article","Asia","Russia","Oblast","Moscow",123,12,111
2013-08-31T11:58:39Z,"Crimson Typhoon","zh","triplets","true","false","true","false","wikipedia","Asia","China","Shanxi","Taiyuan",905,5,900
2013-08-31T12:41:27Z,"Coyote Tango","ja","cancer","true","false","true","false","wikipedia","Asia","Japan","Kanto","Tokyo",1,10,-9
```

*TSV(Delimited)*
```
2013-08-31T01:02:33Z  "Gypsy Danger"  "en"  "nuclear" "true"  "true"  "false" "false" "article" "North America" "United States" "Bay Area"  "San Francisco" 57  200 -143
2013-08-31T03:32:45Z  "Striker Eureka"  "en"  "speed" "false" "true"  "true"  "false" "wikipedia" "Australia" "Australia" "Cantebury" "Syndey"  459 129 330
2013-08-31T07:11:21Z  "Cherno Alpha"  "ru"  "masterYi"  "false" "true"  "true"  "false" "article" "Asia"  "Russia"  "Oblast"  "Moscow"  123 12  111
2013-08-31T11:58:39Z  "Crimson Typhoon" "zh"  "triplets"  "true"  "false" "true"  "false" "wikipedia" "Asia"  "China" "Shanxi"  "Taiyuan" 905 5 900
2013-08-31T12:41:27Z  "Coyote Tango"  "ja"  "cancer"  "true"  "false" "true"  "false" "wikipedia" "Asia"  "Japan" "Kanto" "Tokyo" 1 10  -9
```

请注意，CSV和TSV数据不包含列标题。当您指定要摄取的数据时，这一点就变得很重要。

除了文本格式，Druid还支持二进制格式，比如 [Orc](#orc) 和 [Parquet](#parquet) 格式。

### 定制格式

Druid支持自定义数据格式，可以使用 `Regex` 解析器或 `JavaScript` 解析器来解析这些格式。请注意，使用这些解析器中的任何一个来解析数据都不如编写原生Java解析器或使用外部流处理器那样高效。我们欢迎新解析器的贡献。

### InputFormat

> [!WARNING]
> 输入格式是在0.17.0中引入的指定输入数据的数据格式的新方法。不幸的是，输入格式还不支持Druid支持的所有数据格式或摄取方法。特别是如果您想使用Hadoop接收，您仍然需要使用 [解析器](#parser)。如果您的数据是以本节未列出的某种格式格式化的，请考虑改用解析器。

所有形式的Druid摄取都需要某种形式的schema对象。要摄取的数据的格式是使用[`ioConfig`](../DataIngestion/ingestion.md#ioConfig) 中的 `inputFormat` 条目指定的。

#### JSON

**JSON**
一个加载JSON格式数据的 `inputFormat` 示例：
```
"ioConfig": {
  "inputFormat": {
    "type": "json"
  },
  ...
}
```
JSON `inputFormat` 有以下组件：

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 填 `json` | 是 |
| flattenSpec | JSON对象 | 指定嵌套JSON数据的展平配置。更多信息请参见[flattenSpec](#flattenspec) | 否 |
| featureSpec | JSON对象 | Jackson库支持的 [JSON解析器特性](https://github.com/FasterXML/jackson-core/wiki/JsonParser-Features) 。这些特性将在解析输入JSON数据时应用。 | 否 |


#### CSV
一个加载CSV格式数据的 `inputFormat` 示例：
```
"ioConfig": {
  "inputFormat": {
    "type": "csv",
    "columns" : ["timestamp","page","language","user","unpatrolled","newPage","robot","anonymous","namespace","continent","country","region","city","added","deleted","delta"]
  },
  ...
}
```

CSV `inputFormat` 有以下组件：

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 填 `csv` | 是 |
| listDelimiter | String | 多值维度的定制分隔符 | 否(默认ctrl + A) |
| columns | JSON数组 | 指定数据的列。列的顺序应该与数据列的顺序相同。 | 如果 `findColumnsFromHeader` 设置为 `false` 或者缺失， 则为必填项 |
| findColumnsFromHeader | 布尔 | 如果设置了此选项，则任务将从标题行中查找列名。请注意，在从标题中查找列名之前，将首先使用 `skipHeaderRows`。例如，如果将 `skipHeaderRows` 设置为2，将 `findColumnsFromHeader` 设置为 `true`，则任务将跳过前两行，然后从第三行提取列信息。该项如果设置为true，则将忽略 `columns` | 否（如果 `columns` 被设置则默认为 `false`, 否则为null） |
| skipHeaderRows | 整型数值 | 该项如果设置，任务将略过 `skipHeaderRows`配置的行数 | 否（默认为0） |

#### TSV(Delimited)
```
"ioConfig": {
  "inputFormat": {
    "type": "tsv",
    "columns" : ["timestamp","page","language","user","unpatrolled","newPage","robot","anonymous","namespace","continent","country","region","city","added","deleted","delta"],
    "delimiter":"|"
  },
  ...
}
```
TSV `inputFormat` 有以下组件：

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 填 `tsv` | 是 |
| delimiter | String | 数据值的自定义分隔符 | 否(默认为 `\t`) |
| listDelimiter | String | 多值维度的定制分隔符 | 否(默认ctrl + A) |
| columns | JSON数组 | 指定数据的列。列的顺序应该与数据列的顺序相同。 | 如果 `findColumnsFromHeader` 设置为 `false` 或者缺失， 则为必填项 |
| findColumnsFromHeader | 布尔 | 如果设置了此选项，则任务将从标题行中查找列名。请注意，在从标题中查找列名之前，将首先使用 `skipHeaderRows`。例如，如果将 `skipHeaderRows` 设置为2，将 `findColumnsFromHeader` 设置为 `true`，则任务将跳过前两行，然后从第三行提取列信息。该项如果设置为true，则将忽略 `columns` | 否（如果 `columns` 被设置则默认为 `false`, 否则为null） |
| skipHeaderRows | 整型数值 | 该项如果设置，任务将略过 `skipHeaderRows`配置的行数 | 否（默认为0） |

请确保将分隔符更改为适合于数据的分隔符。与CSV一样，您必须指定要索引的列和列的子集。

#### ORC

> [!WARNING]
> 使用ORC输入格式之前，首先需要包含 [druid-core-extensions](../Development/orc-extensions.md) 

> [!WARNING]
> 如果您正在考虑从早于0.15.0的版本升级到0.15.0或更高版本，请仔细阅读 [从contrib扩展的迁移](../Development/orc-extensions.md#从contrib扩展迁移)。

一个加载ORC格式数据的 `inputFormat` 示例：
```
"ioConfig": {
  "inputFormat": {
    "type": "orc",
    "flattenSpec": {
      "useFieldDiscovery": true,
      "fields": [
        {
          "type": "path",
          "name": "nested",
          "expr": "$.path.to.nested"
        }
      ]
    }
    "binaryAsString": false
  },
  ...
}
```

ORC `inputFormat` 有以下组件：

#### Parquet
#### FlattenSpec
### Parser
#### String Parser
#### Avro Hadoop Parser
#### ORC Hadoop Parser
#### Parquet Hadoop Parser
#### Parquet Avro Hadoop Parser
#### Avro Stream Parser
#### Protobuf Parser
### ParseSpec
#### JSON解析规范
#### JSON Lowercase解析规范
#### CSV解析规范
#### TSV/Delimited解析规范
#### 多值维度
#### 正则解析规范
#### JavaScript解析规范
#### 时间和维度解析规范
#### Orc解析规范
#### Parquet解析规范