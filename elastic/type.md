## 什么是Mapping

Mapping 类似于数据库中的表结构定义schema，它的主要作用是：**用来定义索引中的字段的名称、定义字段的数据类型和定义字段类型的一些其它参数**，比如字符串、数字、布尔字段，倒排索引的相关配置，设置某个字段为不被索引、记录 position 等。每一种数据类型都有对应的使用场景，并且每个文档都有映射，但是在大多数使用场景中，我们并不需要显示的创建映射，因为ES中实现了动态映射。

```json
{
    "name":"jack",
    "age":18,
    "birthDate": "1991-10-05"
}
```

在动态映射的作用下，name会映射成text类型，age会映射成long类型，birthDate会被映射为date类型，映射的索引信息如下。

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "age": {
          "type": "long"
        },
        "birthDate": {
          "type": "date"
        },
        "name": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      }
    }
  }
}
```

自动判断的规则如下：

| 字符串 | 匹配日期，设置为Date。匹配数字，设置为float或者long（默认关闭）。会为字符串类型设置为Text，并增加keyword子字段。 |
| ------ | ------------------------------------------------------------ |
| 布尔值 | boolean                                                      |
| 浮点数 | float                                                        |
| 整数   | long                                                         |
| 对象   | Object                                                       |
| 数组   | 有第一个非空数值的类型所决定                                 |
| 空     | 忽略                                                         |

## Mapping的数据类型

> ES 字段类型类似于 MySQL 中的字段类型，ES 字段类型主要有：核心类型、复杂类型、地理类型以及特殊类型，常见的ELasticSearch数据类型如下：
>
> (ES官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/8.8/mapping-types.html)

| 一级分类 | 二级分类     | 具体类型                                        |
| -------- | ------------ | ----------------------------------------------- |
| 核心类型 | 字符串类型   | ~~string~~，text，keyword                       |
|          | 整数类型     | integer，long，short，byte                      |
|          | 浮点类型     | double，float，half_float，scaled_float         |
|          | 逻辑类型     | boolean                                         |
|          | 日期类型     | date                                            |
|          | 范围类型     | range(Integer_range，long_range，date_range...) |
|          | 二进制类型   | binary (BASE64 的二进制)                        |
| 复合类型 | 数组类型     | array                                           |
|          | 对象类型     | object                                          |
|          | 嵌套类型     | nested                                          |
| 地理类型 | 地理坐标类型 | geo_point                                       |
|          | 地理地图     | geo_shape                                       |
| 特殊类型 | IP类型       | ip                                              |
| ...      | ...          | ...                                             |

###### **字符串类型**(text，keyword)

从ElasticSearch 5.x开始不再支持string，由text和keyword类型替代：

- text 类型适用于需要被全文检索的字段，因为它会分词，例如新闻正文、邮件内容等比较长的文字，text 类型会被 Lucene 分词器（Analyzer）处理为一个个词项，并使用 Lucene 倒排索引存储，**text 字段不能被用于排序**，如果需要使用该类型的字段只需要在定义映射时指定 JSON 中对应字段的 type 为 text。text 类型详细可参考：[Text type family | Elasticsearch Guide [8.8\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.8/text.html)

- keyword 只能通过精确值搜索到，适合简短、结构化字符串，例如主机名、姓名、商品名称等，**可以用于过滤、排序、聚合检索，也可以用于精确查询**。keyword 类型详细可参考：[Keyword type family | Elasticsearch Guide [8.8\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.8/keyword.html)

> 总结：对text类型的字段，会先使用分词器分词，生成倒排索引，用于之后的搜索。对keyword类型的字段，不会分词，搜索时只能精确查找

**关于text类型的常用参数：**

- analyzer：指明该字段用于索引时和搜索时的分析字符串的分词器（使用search_analyzer可覆盖它）。 默认为索引分析器或标准分词器
- fielddata：指明该字段是否可以使用内存中的fielddata进行排序，聚合或脚本编写？默认值为false，可取值true或false。（排序，分组需要指定为true）
- fields：【多数类型】text类型字段会被分词搜索，不能用于排序，而当字段既要能通过分词搜索，又要能够排序，就要设置fields为keyword类型进行聚合排序。
- index：【是否被索引】设置该字段是否可以用于搜索。默认为true，表示可以用于搜索。
- search_analyzer：设置在搜索时，用于分析该字段的分析器，默认是【analyzer】参数的值。
- search_quote_analyzer：设置在遇到短语搜索时，用于分析该字段的分析器，默认是【search_analyzer】参数的值。
- index_options：【索引选项】用于控制在索引过程中哪些信息会被写入到倒排索引中
  - docs：只索引文档号到倒排索引中，但是并不会存储
  - freqs：文档号和关键词的出现频率会被索引，词频用于给文档进行评分，重复词的评分会高于单个次评分
  - positions：文档号、词频和关键词 term 的相对位置会被索引，相对位置可用于编辑距离计算和短语查询(不分词那种)
  - offsets：文档号、词频、关键词 term 的相对位置和该词的起始字符串偏移量

**关于keyword类型的常用参数：**

- eager_global_ordinals：指明该字段是否加载全局序数？默认为false，不加载。 对于经常用于术语聚合的字段，启用此功能是个好主意。
- fields：指明能以不同的方式索引该字段相同的字符串值，例如用于搜索的一个字段和用于排序和聚合的多字段
- ignore_above：不要索引长于此值的任何字符串。默认为2147483647，以便接受所有值
- index：指明该字段是否可以被搜索，默认为true，表示可以被搜索
- index_options：指定该字段应将哪些信息存储在索引中，以便用于评分。默认为docs，但也可以设置为freqs，这样可以在计算分数时考虑术语频率
- norms：在进行查询评分时，是否需要考虑字段长度，默认为false，不考虑
- ignore_above：默认值是256，该参数的意思是，当字段文本的长度大于指定值时，不会被索引，但是会存储。即当字段文本的长度大于指定值时，聚合、全文搜索都查不到这条数据。ignore_above 最大值是 32766 ，但是要根据场景来设置，比如说中文最大值 应该是设定在10922 。

###### **数字类型**(integer，long，short，byte，double，float，half_float，scaled_float)

ES支持的数字类型有整型数字类型，integer类型、long类型、short类型、byte类型。浮点型数字类型 double类型、 float类型、 half_float类型、 scaled_float类型这类数据类型都是以确切值索引的，可以使用term查询精确匹配。数字类型的字段在满足需求的前提下应当尽量选择范围较小的数据类型，字段长度越短，搜索效率越高，对于浮点数，可以优先考虑使用 scaled_float 类型，该类型可以通过缩放因子来精确浮点数，例如 12.34 可以转换为 1234 来存储。

- long带符号的64位整数，最小值-263，最大值263-1
- integer带符号的32位整数，最小值-231，最大值231^-1
- short带符号的16位整数，最小值-32768，最大值32767
- byte带符号的8位整数，最小值-128，最小值127
- double双精度64位IEEE 754 浮点数
- float单精度32位IEEE 754 浮点数
- half_float半精度16位IEEE 754 浮点数
- scaled_float带有缩放因子的缩放类型浮点数，依靠一个long数字类型通过一个固定的(double类型)缩放因数进行缩放

###### **日期类型**(date)

在 ES 中日期可以为以下形式：格式化的日期字符串，例如 2020-03-17 00:00、2020/03/17时间戳（和 1970-01-01 00:00:00 UTC 的差值），单位毫秒或者秒即使是格式化的日期字符串，ES 底层依然采用的是时间戳的形式存储。

日期类型一般会结合一个mapping参数来使用：format——自定义的日期格式，默认：strict_date_optional_time || epoch_millis。

```json
PUT test_index
{
  "mappings": {
    "properties": {
      "date1": {
        "type": "date"
      },
      "date2": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "date3": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy/MM/dd||epoch_millis"
      }
    }
  }
}
#添加2条数据
POST test_index/_doc
{
  "dateTimeFormat": "2015-01-01 12:10:30"
}


PUT test_index/_doc/1
{
  "date1": 1577808000000,
  "date2": "2020-01-01 00:00:00",
  "date3": "2020/01/01"
}
PUT test_index/_doc/2
{
  "date1": 1577808000001,
  "date2": "2020-01-01 00:00:01",
  "date3": "2020/01/01"
}
#查询测试
GET test_index/_search
{
  "sort": [
    {
      "date2": {
        "order": "desc"
      }
    }
  ]
}
```

###### 布尔类型(boolean)

JSON 文档中同样存在布尔类型，不过 JSON 字符串类型也可以被 ES 转换为布尔类型存储，前提是字符串的取值为 true 或者 false，布尔类型常用于检索中的过滤条件。

###### 二进制类型(binary)

二进制类型 binary 接受 BASE64 编码的字符串，默认 store 属性为 false，并且不可以被搜索。

###### 范围类型(range)

范围类型可以用来表达一个数据的区间，所以会用到gt、gte、lt、lte…等逻辑表示符。可以分为6种：integer_range、float_range、long_range、double_range、date_range以及ip_range。

- integer_range，带符号的32位整数区间，最小值 -2的31次方，最大值2的31次方-1
- long_range，带符号的64位整数区间，最小值-2的63 次方，最小值2的63次方-1
- float_range，单精度32位IEEE 754浮点数区间
- double_range，双精度64位IEEE 754浮点数区间
- date_range，日期值范围，表示为系统纪元以来经过的无符号64位整数毫秒
- ip_range，支持IPv4或IPv6（或混合）地址ip值范围

```json
PUT test_index
{
  "mappings": {
    "properties": {
      "data1": {
        "type": "integer_range"
      },
      "data2": {
        "type": "float_range"
      },
      "data3": {
        "type": "date_range",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "data4": {
        "type": "ip_range"
      }
    }
  }
}
```

```json
PUT test_index/_doc/1
{
  "date1": {
    "gte": 100,
    "lte": 200
  },
  "date2": {
    "gte": 21.21,
    "lte": 22
  },
  "date3": {
    "gte": "2020-01-01 00:00:00",
    "lte": "2020-01-02 00:00:00"
  },
  "date4": {
    "gte": "192.168.192.10",
    "lte": "192.168.192.11"
  }
}
```

###### 对象类型(object)

对象类型即一个JSON对象，JSON 字符串允许嵌套对象，所以一个文档可以嵌套多个、多层对象。可以通过对象类型来存储二级文档，不过由于 Lucene 并没有内部对象的概念，所以ES 会将原 JSON 文档扁平化，例如有下面这样的文档：

```json
PUT test_index
{
  "mappings": {
    "properties": {
      "user": {
        "type": "object"
      }
    }
  }
}

PUT test_index/_doc/1
{
  "user": {
    "first": "zhang",
    "last": "san"
  }
}
```

上面我们看到都是一个个分开的字段，而实际上 ES 会将其转换为以下格式，并通过 Lucene 存储，即使 name 是 object 类型：

```json	
{
    "user.first": "zhang",
    "user .last": "san"
}
```

所以我们在进行条件搜索是也必须使用这种方式：

```json	
GET test_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "user.first": "zhang"
          }
        }
      ]
    }
  }
}
```

###### 嵌套类型(nested)

嵌套类型可以看成是一个特殊的对象类型，在文档属性是一个对象数组时使用，它允许对象数组彼此独立地编制索引和查询，例如文档：

```json
{
    "group": "users",
    "user": [
        {"first":"zhang","last":"san"},
        {"first":"li","last":"si"},
        {"first":"wang","last":"wu"}
    ]
}
```

username 字段是一个 JSON 数组，并且每个数组对象都是一个 JSON 对象。如果将 username 设置为对象类型，那么 ES 会将其转换为：

```none
{
    "group": "users",
    "user.first": ["zhang","li","wang"],
    "user.last": ["san","si","wu"]
}
```

可以看出转换后的 JSON 文档中 first 和 last 的关联丢失了，如果尝试搜索 first 为 li，last 为 wu 的文档，那么成功会检索出上述文档，但是 li 和 wu 在原 JSON 文档中并不属于同一个 JSON 对象，应当是不匹配的，即检索不出任何结果。而嵌套类型就是为了解决这种问题的，嵌套类型将数组中的每个 JSON 对象作为独立的隐藏文档来存储，每个嵌套的对象都能够独立地被搜索。

测试查询：

```json
DELETE test_index

PUT test_index
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested"
      }
    }
  }
}

PUT test_index/_doc/1
{
  "username": [
    {
      "first": "li",
      "last": "si"
    },
    {
      "first": "wang",
      "last": "wu"
    }
  ]
}

#查询测试
GET test_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "user.first": "li"
          }
        },
        {
          "match": {
            "user.last": "wu"
          }
        }
      ]
    }
  }
}

# 有效查询
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path":"user",
            "query": {
              "bool": {
                "must":[
                  {
                    "match":{
                      "user.name":"li"
                    }
                  },
                  {
                     "match":{
                      "user.last":"wu"
                    }
                  }
                ]
              }
            }
          }
        },
        {
          "match": {
            "user.last": "wu"
          }
        }
      ]
    }
  }
}

```



## Mapping的主要参数

### mapping组成

一个mapping主要有两部分组成：metadata和mapping：

- metadata元数据字段用于自定义如何处理文档关联的元数据。例如：
  - _index：用于定义document属于哪个index
  - _type：类型，已经移除的概念
  - _id：document的唯一id
  - _source：存放原始的document数据
  - _size：_source字段中存放的数据的大小
- mapping中包含的field，包含字段的类型和参数。本文主要介绍的mapping参数就需要在field中去定义。例如：
  - type：设置字段对应的类型，常见的有text，keyword等
  - analyzer：指定一个用来文本分析的索引或者搜索text字段的分析器 应用于索引以及查询

### mapping参数

> 字段映射参数详细说明，更详细的请参考官方文档：[Mapping parameters | Elasticsearch Guide [8.8\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.8/mapping-params.html)

主要参数如下：

- **analyzer：**只能用于text字段，用于根据需求设置不通的分词器，默认是ES的标准分词

- **boost：**默认值为1。用于设置字段的权重，主要应用于查询时候的评分

- **coerce：**默认是true。主要用于清理脏数据来匹配字段对应的类型。例如字符串“5”会被强制转换为整数，浮点数5.0会被强制转换为整数
- **copy_to**：能够把几个字段拼成一个字段。老字段和新组成的字段都可以查询

- **doc_values：**默认值为true。Doc Values和倒排索引同时生成，本质上是一个序列化的 列式存储。列式存储适用于聚合、排序、脚本等操作，也很适合做压缩。如果字段不需要聚合、排序、脚本等操作可以关闭掉，能节省磁盘空间和提升索引速度。

- **dynamic**：

  默认值为true。默认如果插入的document字段中有mapping没有的，会自动插入成功，并自动设置新字段的类型；如果一个字段中插入一个包含多个字段的json对象也会插入成功。但是这个逻辑可以做限制：

  - ture: 默认值，可以动态插入
  - false：数据可写入但是不能被索引分析和查询，但是会保存到_source字段。
  - strict：无法写入

- **eager_global_ordinals：**默认值为false。设置每refresh一次就创建一个全局的顺序映射，用于预加载来加快查询的速度。需要消耗一定的heap。
- **enabled：**默认值为true。设置字段是否索引分析。如果设置为false，字段不对此字段索引分析和store，会导致此字段不能被查询和聚合，但是字段内容仍然会存储到_source中。

- **fielddata：**默认值为false，只作用于text字段。默认text字段不能排序，聚合和脚本操作，可以通过开启此参数打开此功能。但是会消耗比较大的内存。

- **fields：**可以对一个字段设置多种索引类型，例如text类型用来做全文检索，再加一个keyword来用于做聚合和排序。
- **format：**用于date类型。设置时间的格式。具体见https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html

- **ignore_above：**默认值为256，作用于keyword类型。指示该字段的最大索引长度（即超过该长度的内容将不会被索引分析），对于超过ignore_above长度的字符串，analyzer不会进行索引分析，所以超过该长度的内容将不会被搜索到。注意：keyword类型的字段的最大长度限制为32766个UTF-8字符，text类型的字段对字符长度没有限制
- **ignore_malformed：**默认为false。插入新document的时候，是否忽略字段的类型，默认字段类型必须和mapping中设置的一样

- **index_options**：

  默认值为positions，只作用于text字段。控制将哪些信息添加到倒排索引中以进行搜索和突出显示。有4个选项：

  - docs 添加文档号
  - freqs 添加文档号和次频
  - positions 添加文档号，词频，位置
  - offsets 添加文档号，词频，位置，偏移量

- **index：**默认值为true。设置字段是否会被索引分析和可以查询
- **meta：**可以给字段设置metedata字段，用于标记等

- **normalizer：**可以对字段做一些标准化规则，例如字符全部大小写等

- **norms：**默认值为true。默认会存储了各种规范化因子，在查询的时候使用这些因子来计算文档相对于查询的得分，会占用一部分磁盘空间。如果字段不用于检索，只是过滤，查询等精确操作可以关闭。
- **null_value：**null_value意味着无法索引或搜索空值。当字段设置为 null , [] ,和 [null]（这些null的表示形式都是等价的），它被视为该字段没有值。通过设置此字段，可以设置控制可以被索引和搜索。

- **properties：**如果这个字段有嵌套属性，包含了多个子字段。需要用到properties

- **search_analyzer：**默认值和analyzer相同。在查询时，先对要查询的text类型的输入做分词，再去倒排索引搜索，可以通过这个设置查询的分析器为其它的，默认情况下，查询将使用analyzer字段制定的分析器，但也可以被search_analyzer覆盖

- **similarity**：

  用于设置document的评分模型，有三个：

  - BM25:lucene的默认评分模型
  - classic:TF/IDF评分模型
  - boolean:布尔评分模型

- **store：**默认为false，lucene不存储原始内容，但是_source仍然会存储。这个属性其实是lucene创建字段时候的一个选项，表明是否要单独存储原始值（_source字段是elasticsearch单独加的和store没有关系）。如果字段比较长，从_source中获取损耗比较大，可以关闭_source存储，开启store。
- **term_vector：** 用于存储术语的规则。默认值为no，不存储向量信息.

```json
创建一个索引，并且指定索引类型

PUT /my_test
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      }
    }
  }
}

索引已经存在，在原有的索引上增加一个字段与类型
PUT /my_test/_mapping
{
  "properties": {
    "count": {
      "type": "long"
    }
  }
}
```

注意：手工创建mapping时，字段的mapping只能创建，无法修改。