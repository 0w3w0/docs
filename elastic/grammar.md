#### Query String

###### 创建索引

```DSL
PUT /test_user
```

######  删除索引

```shell
DELETE /test_user
```

###### 向索引添加数据

```shell
PUT /test_user/_doc/1
{
  "name":"李四",
  "age":18,
  "email":"123@qq.com"
}
```

###### 删除索引中指定的数据

```shell
DELETE /test_user/_doc/1
```

###### 查询索引下的所有数据

```shell
GET /test_user/_search
```

###### URL中带参数查询

```shell
GET /test_user/_search?q=age:18
```

####  DSL查询

###### match_all 查询所有数据

```shell
GET /test_user/_search
{
  "query": {
    "match_all": {}
  }
}
```

######  match

match是模糊匹配查询，根据分词器(如果创建mapping没有指定分词器，Es将会采取默认的分词器:standard,standard分词将会把匹配的词组分成单个的字,而不是短语)将指定的query查询的语句进行分词匹配。

```shell
GET /test_user/_search
{
  "query": {
    "match": {
      "name": "张三"
    }
  }
}
```

###### term

英文含义表示是:精确的意思，在term查询中，表示做的是精确查询，整词匹配，不会对所匹配项进行拆分。直接以整词进行匹配，如果能查询到就命中该文档

```shell
#term 会直接对关键词进行查找
#如： my name is bl 会直接当成一个词“my name is bl”
#    match 会分成4个词
GET /test_user/_search
{
  "query": {
    "term": {
      "age": {
        "value": 18
      }
    }
  }
}
```

###### **terms**

terms和term的区别就是terms允许匹配多个值，而term只允许匹配一个值,在进行多值匹配的场景中可以使用terms,terms匹配到其中任何一个值就会认为整个文档是匹配的,terms多个值如果多个都匹配会返回所有文档

```json
查询age为24、66的任一值，查到就返回文档
{
    'query': {
        'terms': {
            'age': ['24','36']
        }
    }
}
```



######  multi_match

```shell
GET /test_user/_search
{
  "query": {
    "multi_match": {
      "query": "1", # 查询的内容
      "fields": ["name","email"] # 
    }
  }
}
```

###### sort 排序

```shell
GET /test_user/_search
{
  "sort": [
    {
      "age": {
        "order": "asc"
      }
    }
  ]
}
```

###### 返回指定的字段(_source)

```shell
GET /test_user/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["name","age"]
}
```

###### 分页查询 from size

```shell
# from，起始条数 总共多少条数据
GET /test_index/_search
{
  "from": 0,
  "size": 10
}
```

###### range

range表示一个区间范围查找,这个范围可以是日期或者数值，ES的range比较灵活和明确,可以指定两个边界是否包含，通过参数include_lower:true 、include_upper:true来控制

```json
查询age为20到30区间所有文档,
include_lower表示是否包含边界最小值(true表示包含),
include_upper表示是否包含边界最大值(true表示包含,false表示不包含)
 
{
    'query': {
        'range': {
            'age': {
                'from': 20,
                'to': 30,
                'include_lower':true,
                'include_upper':false
                }
        }
    }
}
```

```json
查询log_time对应时间区间内的所有文档,
gt/gt表示起始时间，gte包含起始时间点,gt不包含起始时间点,
lt/lte表示结束时间，lte包结束时间点,lt不包含结束时间点
 
{
    'query': {
        'range': {
                'log_time': {
                    'gte': '2022-04-02 00:00:00',
                    'lt': '2022-04-02 23:00:00',
                    'format': 'yyyy-MM-dd HH:mm:ss',
                }
            }
    }
}
```



###### fuzzy

fuzzy查询的时候,不会根据分词器匹配,只会进行拆分,比如查询的是"海zei王",在分词器下(也就是match中)是无法匹配到单个词的，因为它不是一个短语,但是在fuzzy中是可以匹配的，并且fuzzy支持模糊和一定的容错查询匹配,因为它做的是匹配词的拆分,并不是短语。

```json
查询索引中name为海zei王的文档：
{
    'query': {
        'fuzzy': {
            'name': '海zei王'
        }
    }
}
```

###### **regexp**

正则表达式匹配，该匹配模式下我们可以按照正则表达式的符号去匹配具体的值,比如name字段，可以包含有.和*去正则匹配具体的值，?表示任一字符，*表示所有字符，还有其他的正则符号都可以使用，详情参考[Regexp query | Elasticsearch Guide [8.8\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.8/query-dsl-regexp-query.html#regexp-syntax)

```json
匹配手机号以1到9之前的开头，并且第二位是3最后一位是0或者1的手机号的文档：
{
    'query': {
        'regexp': {
            'phone': {
                'value':'[1-9]3.*[0-1]'
            }
        }
    }
}
```

###### **wildcard**

通配符匹配，wildcard和regexp类似,不过它们也有不同之处。regexp的实际匹配能力要大于wildcard，在进行简单的匹配时候，比如名字的*或者?的简单普通匹配,建议使用wildcard而不是regexp,wildcard的效率要高于regexp，regexp可以实现更为复杂的场景,但是效率低一些,通俗的说wildcard是regexp的简化版本.
```json
采用通配符匹配手机的电话号
{
    'query': {
        'wildcard': {
            'phone': '13643091*'
        }
    }
}
```



###### match_phrase 短语搜索

```shell
# match_phrase 将数据分词后进行搜索的
#    1、目标文档需要包含分词后的所有词
#    2、目标文档还要保持这些词的相对顺序和文档中的一致
#    3、只有当这三个条件满足，才会命中文档！
# 而不像 match那样，将"li zhang san"分成三个词进行探索
GET /test_user/_search
{
  "query": {
    "match_phrase": {
      "name": "li zhang san"
    }
  }
}
```

#### bool查询与过滤

在使用布尔查询时，需要将查询的条件写在`bool`查询语句中，且同个`bool`查询语句可以有多个不同的条件。

```json
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "age": 20
        }
      },
      "must_not": {
        "term": {
          "gender": "male"
        }
      }
    }
  }
}
```

例如该查询运行后，将返回`age`的值为`20`且`gender`的值不为`"male"`的文档。

##### must

返回的文档必须满足must子句的条件,包含多个条件时类似于SQL中的`AND`，并且参与计算分值。

当`must`查询只包括一个查询条件时，可在DSL中使用JSON对象的形式表示，例如以下示例

```json
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "age": 20
        }
      }
    }
  }
}

该查询等同于下面对应的SQL语句：
SELECT * FROM xxx WHERE age = 20;
```

使用`must`时可以同时指定多个查询条件，在DSL中它以数组的形式表示，效果类似于SQL中的`AND`运算。例如下面的例子：

```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "age": 20 } },
        { "term": { "gender": "male" } }
      ]
    }
  }
}

该查询等同于下面的SQL语句：
SELECT * FROM xxx WHERE age = 20 AND gender = "male";
```

##### filter

返回的文档必须满足filter子句的条件，效果与使用`must`相同。但是不会像must一样，`filter`查询不参与查询结果的分值计算，它返回的文档的分值始终为0。

`filter`的使用场景适合于过滤不需要的文档，但又不影响最终计算的得分。

##### should

返回的文档可能满足should子句的条件。在一个Bool查询中，如果没有must或者filter，有一个或者多个should子句，那么只要满足一个就可以返回，包含多个条件时类似于SQL中的`OR`。`minimum_should_match`参数定义了至少满足几个子句。

```json
{
  "query": {
    "bool": {
      "should": [
        { "term": { "age": 20 } },
        { "term": { "gender": "male" } },
        { "range": { "height": { "gte": 170 } } },
      ]
    }
  }
}

该查询等同于下面对应的SQL语句：
SELECT * FROM xxx WHERE age = 20 OR gender = "male" or height >= 170;
```

`should`查询与SQL中的`OR`运算较为不同的一点是，`should`查询可以使用`minimum_should_match`参数指定至少需要满足几个条件。例如下面的例子中，查询的结果需要满足两个或两个以上的查询条件：

```json
{
  "query": {
    "bool": {
      "should": [
        { "term": { "age": 20 } },
        { "term": { "gender": "male" } },
        { "term": { "height": 170 } },
      ],
      "minimum_should_match": 2
    }
  }
}
```

在同一个`bool`语句中若不存在`must`或`filter`时，`minimum_should_match`默认的值为1，即至少要满足其中一个条件；但若有其它`must`或`filter`存在时，`minimum_should_match`默认值为0。例如下面的查询，所有返回的文档`age`值必定为20，但其中可能包括有`status`值不为`"active"`的文档。若需要二者同时生效，可入上面例子中一样在`bool`查询中增加一个参数`"minimum_should_match": 1`。

```json
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "age": 20
        },
      },
      "should": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

##### must_nout

返回的文档必须不满足must_not定义的条件，类似于SQL中的`NOT`，且返回的结果的分值都为`0`。

> 如果一个查询既有filter又有should，那么至少包含一个should子句。

```json
{
  "query": {
    "bool": {
      "must_not": [
        { "term": { "age": 20 } },
        { "term": { "gender": "male" } }
      ]
    }
  }
}

该查询等同于下面的SQL查询语句（由于MySQL不支持下面语句使用NOT，于是改写为使用!=实现）：
SELECT * FROM xxx WHERE age != 20 AND gender != "male";
```

##### 布尔组合查询

在上面的示例(本文第一个示例代码）中，我们提到过同一个`bool`下可以存在多个不同的查询条件，如该查询等同于下面的SQL语句：

```sql
SELECT * FROM xxx WHERE age = 20 AND gender != "male";
```

另外，我们也可以在各个查询中进行嵌套查询。但需要注意的是，布尔查询必须包含在`bool`查询语句中，所以在嵌套查询中必须在内部再次使用`bool`查询语句。

```json
{
  "query": {
    "must": [
      {
        "bool": {
          "should": [
            { "term": { "age": 20 } },
            { "term": { "age": 25 } }
          ]
        }
      },
      {
        "range": {
          "level": {
            "gte": 3
          }
        }
      }
    ]
  }
}

该查询语句等同于以下SQL语句：
SELECT * FROM xxx WHERE (age = 20 OR age = 25) AND level >= 3;
```



#### bool.filter的分值计算

在filter子句查询中，分值将会都返回0。分值会受特定的查询影响。

比如，下面三个查询中都是返回所有status字段为active的文档

第一个查询，所有的文档都会返回0:

```shell
GET _search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

下面的bool查询中包含了一个match_all，因此所有的文档都会返回1

```shell
GET _search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

constant_score与上面的查询结果相同，也会给每个文档返回1：

```json
GET _search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

#### 使用named query给子句添加标记

如果想知道到底是bool里面哪个条件匹配，可以使用named query查询：

```shell
{
    "bool" : {
        "should" : [
            {"match" : { "name.first" : {"query" : "shay", "_name" : "first"} }},
            {"match" : { "name.last" : {"query" : "banon", "_name" : "last"} }}
        ],
        "filter" : {
            "terms" : {
                "name.last" : ["banon", "kimchy"],
                "_name" : "test"
            }
        }
    }
}
```

