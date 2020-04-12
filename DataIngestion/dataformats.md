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
> 使用ORC输入格式之前，首先需要包含 [druid-orc-extensions](../Development/orc-extensions.md) 

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

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 填 `orc` | 是 |
| flattenSpec | JSON对象 | 指定嵌套JSON数据的展平配置。更多信息请参见[flattenSpec](#flattenspec) | 否 |
| binaryAsString | 布尔类型 | 指定逻辑上未标记为字符串的二进制orc列是否应被视为UTF-8编码字符串。 | 否（默认为false） |

#### Parquet

> [!WARNING]
> 使用Parquet输入格式之前，首先需要包含 [druid-parquet-extensions](../Development/parquet-extensions.md) 

一个加载Parquet格式数据的 `inputFormat` 示例：
```
"ioConfig": {
  "inputFormat": {
    "type": "parquet",
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

Parquet `inputFormat` 有以下组件：

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 填 `parquet` | 是 |
| flattenSpec | JSON对象 | 定义一个 [flattenSpec](#flattenspec) 从Parquet文件提取嵌套的值。注意，只支持"path"表达式（'jq'不可用）| 否（默认自动发现根级别的属性） |
| binaryAsString | 布尔类型 | 指定逻辑上未标记为字符串的二进制orc列是否应被视为UTF-8编码字符串。 | 否（默认为false） |

#### FlattenSpec

`flattenSpec` 位于 `inputFormat` -> `flattenSpec` 中，负责将潜在的嵌套输入数据（如JSON、Avro等）和Druid的平面数据模型之间架起桥梁。 `flattenSpec` 示例如下：
```
"flattenSpec": {
  "useFieldDiscovery": true,
  "fields": [
    { "name": "baz", "type": "root" },
    { "name": "foo_bar", "type": "path", "expr": "$.foo.bar" },
    { "name": "first_food", "type": "jq", "expr": ".thing.food[1]" }
  ]
}
```
> [!WARNING]
> 概念上，输入数据被读取后，Druid会以一个特定的顺序来对数据应用摄入规范： 首先 `flattenSpec`(如果有)，然后 `timestampSpec`, 然后 `transformSpec` ,最后是 `dimensionsSpec` 和 `metricsSpec`。在编写摄入规范时需要牢记这一点

展平操作仅仅支持嵌套的 [数据格式](dataformats.md), 包括：`avro`, `json`, `orc` 和 `parquet`。

`flattenSpec` 有以下组件：

| 字段 | 描述 | 默认值 |
|-|-|-|
| useFieldDiscovery | 如果为true，则将所有根级字段解释为可用字段，供 [`timestampSpec`](../DataIngestion/ingestion.md#timestampSpec)、[`transformSpec`](../DataIngestion/ingestion.md#transformSpec)、[`dimensionsSpec`](../DataIngestion/ingestion.md#dimensionsSpec) 和 [`metricsSpec`](../DataIngestion/ingestion.md#metricsSpec) 使用。<br><br> 如果为false，则只有显式指定的字段（请参阅 `fields`）才可供使用。 | true |
| fields | 指定感兴趣的字段及其访问方式, 详细请见下边 | `[]` |

**字段展平规范**

`fields` 列表中的每个条目都可以包含以下组件：

<table>
  <thead>
    <tr>
      <td>字段</td>
      <td>描述</td>
      <td>默认值</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>type</td>
      <td>
        可选项如下：
        <ul>
          <li><code>root</code>, 引用记录根级别的字段。只有当<code>useFieldDiscovery</code> 为false时才真正有用。</li>
          <li><code>path</code>, 引用使用 <a href="https://github.com/json-path/JsonPath">JsonPath</a> 表示法的字段，支持大多数提供嵌套的数据格式，包括<code>avro</code>,<code>csv</code>, <code>json</code> 和 <code>parquet</code></li>
          <li><code>jq</code>, 引用使用 <a href="https://github.com/eiiches/jackson-jq">jackson-jq</a> 表示法的字段， 仅仅支持<code>json</code>格式</li>
        </ul>
      </td>
      <td>none(必填)</td>
    </tr>
    <tr>
      <td>name</td>
      <td>展平后的字段名称。这个名称可以被<code>timestampSpec</code>, <code>transformSpec</code>, <code>dimensionsSpec</code>和<code>metricsSpec</code>引用</td>
      <td>none(必填)</td>
    </tr>
    <tr>
      <td>expr</td>
      <td>用于在展平时访问字段的表达式。对于类型 `path`，这应该是 <a href="https://github.com/json-path/JsonPath">JsonPath</a>。对于 `jq` 类型，这应该是 <a href="https://github.com/eiiches/jackson-jq">jackson-jq</a> 表达式。对于其他类型，将忽略此参数。</td>
      <td>none(对于 `path` 和 `jq` 类型的为必填)</td>
    </tr>
  </tbody>
</table>

**展平操作的注意事项**

* 为了方便起见，在定义根级字段时，可以只将字段名定义为字符串，而不是JSON对象。例如 `{"name": "baz", "type": "root"}` 等价于 `baz`
* 启用 `useFieldDiscovery` 只会在根级别自动检测与Druid支持的数据类型相对应的"简单"字段, 这包括字符串、数字和字符串或数字列表。不会自动检测到其他类型，其他类型必须在 `fields` 列表中显式指定
* 不允许重复字段名（`name`）, 否则将引发异常
* 如果启用 `useFieldDiscovery`，则将跳过与字段列表中已定义的字段同名的任何已发现字段，而不是添加两次
* [http://jsonpath.herokuapp.com/](http://jsonpath.herokuapp.com/) 对于测试 `path`-类型表达式非常有用
* jackson jq支持完整 [`jq`](https://stedolan.github.io/jq/)语法的一个子集。有关详细信息，请参阅 [jackson jq](https://github.com/eiiches/jackson-jq) 文档

### Parser

> [!WARNING]
> parser在 [本地批任务](native.md), [Kafka索引任务](kafka.md) 和 [Kinesis索引任务](kinesis.md) 中已经废弃，在这些类型的摄入方式中考虑使用 [inputFormat](#数据格式)

该部分列出来了所有默认的以及核心扩展中的解析器。对于社区的扩展解析器，请参见 [社区扩展列表](../Development/extensions.md#社区扩展)

#### String Parser

`string` 类型的解析器对基于文本的输入进行操作，这些输入可以通过换行符拆分为单独的记录, 可以使用 [`parseSpec`](#parsespec) 进一步分析每一行。

| 字段 | 类型 | 描述 | 是否必须 |
|-|-|-|-|
| type | string | 一般是 `string`, 在Hadoop索引任务中为 `hadoopyString` | 是 |
| parseSpec | JSON对象 | 指定格式，数据的timestamp和dimensions | 是 |

#### Avro Hadoop Parser

> [!WARNING]
> 需要添加 [druid-avro-extensions](../Development/avro-extensions.md) 来使用 Avro Hadoop解析器

该解析器用于 [Hadoop批摄取](hadoopbased.md)。在 `ioConfig` 中，`inputSpec` 中的 `inputFormat` 必须设置为 `org.apache.druid.data.input.avro.AvroValueInputFormat`。您可能想在 `tuningConfig` 中的 `jobProperties` 选项设置Avro reader的schema， 例如：`"avro.schema.input.value.path": "/path/to/your/schema.avsc"` 或者 `"avro.schema.input.value": "your_schema_JSON_object"`。如果未设置Avro读取器的schema，则将使用Avro对象容器文件中的schema，详情可以参见 [avro规范](http://avro.apache.org/docs/1.7.7/spec.html#Schema+Resolution)

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 应该填 `avro_hadoop` | 是 |
| parseSpec | JSON对象 | 指定数据的时间戳和维度。应该是“avro”语法规范。| 是 |

Avro parseSpec可以包含使用"root"或"path"字段类型的 [flattenSpec](#flattenspec)，这些字段类型可用于读取嵌套的Avro记录。Avro当前不支持“jq”字段类型。

例如，使用带有自定义读取器schema文件的Avro Hadoop解析器：
```
{
  "type" : "index_hadoop",
  "spec" : {
    "dataSchema" : {
      "dataSource" : "",
      "parser" : {
        "type" : "avro_hadoop",
        "parseSpec" : {
          "format": "avro",
          "timestampSpec": <standard timestampSpec>,
          "dimensionsSpec": <standard dimensionsSpec>,
          "flattenSpec": <optional>
        }
      }
    },
    "ioConfig" : {
      "type" : "hadoop",
      "inputSpec" : {
        "type" : "static",
        "inputFormat": "org.apache.druid.data.input.avro.AvroValueInputFormat",
        "paths" : ""
      }
    },
    "tuningConfig" : {
       "jobProperties" : {
          "avro.schema.input.value.path" : "/path/to/my/schema.avsc"
      }
    }
  }
}
```

#### ORC Hadoop Parser

> [!WARNING]
> 需要添加 [druid-orc-extensions](../Development/orc-extensions.md) 来使用ORC Hadoop解析器

> [!WARNING]
> 如果您正在考虑从早于0.15.0的版本升级到0.15.0或更高版本，请仔细阅读 [从contrib扩展的迁移](../Development/orc-extensions.md#从contrib扩展迁移)。

该解析器用于 [Hadoop批摄取](hadoopbased.md)。在 `ioConfig` 中，`inputSpec` 中的 `inputFormat` 必须设置为 `org.apache.orc.mapreduce.OrcInputFormat`。

| 字段 | 类型 | 描述 | 是否必填 |
|-|-|-|-|
| type | String | 应该填 `orc` | 是 |
| parseSpec | JSON对象 | 指定数据(`timeAndDim` 和 `orc` 格式)的时间戳和维度和一个`flattenSpec`（`orc`格式）| 是 |

解析器支持两种 `parseSpec` 格式： `orc` 和 `timeAndDims`

`orc` 支持字段的自动发现和展平（如果指定了 [flattenSpec](#flattenspec)。如果未指定展平规范，则默认情况下将启用 `useFieldDiscovery`。如果启用了 `useFieldDiscovery`，则指定`dimensionSpec` 是可选的：如果提供了 `dimensionSpec`，则它定义的维度列表将是摄取维度的集合，如果缺少发现的字段将构成该列表。

`timeAndDims` 解析规范必须通过 `dimensionSpec` 指定哪些字段将提取为维度。

支持所有 [列类型](https://orc.apache.org/docs/types.html) ，但 `union` 类型除外。`list` 类型的列（如果用基本类型填充）可以用作多值维度，或者可以使用 [flattenSpec](#flattenspec) 表达式提取特定元素。同样，可以用同样的方式从 `map` 和 `struct` 类型中提取基本字段。自动字段发现将自动为每个（非时间戳）基本类型或基本类型 `list` 以及 `flattenSpec` 中定义的任何展平表达式创建字符串维度。

**Hadoop job属性**

像大多数Hadoop作业，
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