# DSL
Elasticsearch提供了DSL(Domain Specific Language)来定义查询

## 常见的查询类型
1. 全量查询：查询出所有数据(分页)，一般用于测试
   1. match_all
2. 全文检索查询：利用分词器对用户输入内容进行分词，然后去倒排索引库中匹配，常用于搜索框搜索
   1. match
   2. multi_match
3. 精确查询：根据精确词条查找数据，一般是查找keyword、数值、日期、boolean等数据，不会对查询内容分词
   1. ids：根据id查询
   2. range：根据范围查询
   3. term：根据数据的值进行查询
4. 地理查询：根据经纬度查询
   1. geo_distance
   2. geo_bounding_box
5. 复合(compound)查询：复合查询可以组合上述查询，实现逻辑更复杂的查询
   1. bool：按照逻辑条件组合查询条件
   2. function_score：根据相关度权重组合查询条件

## 查询基本语法
```json
GET /索引库名/_search
{
    "query": {
        "查询类型": {
            "查询条件": "条件值"
        }
    }
}
```

## 基础返回结构
```json
{
    "took" : 0, // 查询用时，单位毫秒
    "timed_out" : false, // 是否超时
    "_shards" : { // 集群分片情况
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : { // 查询结果
        "total" : { // 结果集数量
            "value" : 500,
            "relation" : "eq"
        },
        "max_score" : 1.0, // 查询得分最高的文档得分
        "hits" : [ // 文档内容
        {
            "_index" : "demo",
            "_type" : "_doc",
            "_id" : "7ng834wBVrLiaeonIHbG",
            "_score" : 1.0,
            "_source" : {
                // 文档内容
            }
        }
        ...
        ]
    }
}

```

## match_all
match_all没有查询条件，默认情况下进行分页
```json
{
    "query": {
        "match_all": {
        }
    }
}
```

## match
需要指定查询文档中哪个字段(一般是text类型的字段)，和查询内容，查询结果按相关度排序
```json
{
    "query": {
        "match": {
            "字段名": "查询内容"
        }
    }
}
```

## multi_match
与match相似，支持多个字段同时进行查询，查询结果按相关度排序
```json
{
    "query": {
        "multi_match": {
            "query": "查询内容",
            "fields": ["字段名1", "字段名2"]
        }
    }
}
```

## term
根据词条进行精确值查询
```json
{
    "query": {
        "term": {
            "字段名": {
                "value": "查询值"
            }
        }
    }
}
```

## range
根据值的范围查询，一般用于数值、keyword(字典序)
```json
{
    "query": {
        "range": {
            "字段名": {
                "操作符1": "查询值1",
                "操作符2": "查询值2"
            }
        }
    }
}
```

## geo_bounding_box
查询geo_point值落在某个矩形范围内的所有文档
```json
{
    "query": {
        "geo_bounding_box": {
            "字段名": {
                "top_left": {
                    "lat": 00.0,
                    "lon": 00.0,
                },
                "bottom_right": {
                    "lat": 00.0,
                    "lon": 00.0,
                }
            }
        }
    }
}
```

## geo_distance
查询geo_point值落在某个圆形范围内的所有文档
```json
{
    "query": {
        "geo_distance": {
            "distance": "距离，单位m,km",
            "字段名": "中心点坐标"
        }
    }
}
```

## function_score
算分函数查询，可以控制文档相关性算分，控制文档排名，比如搜索结果竞价
```json
{
    "query": {
        "function_score": {
            "query": {
                // 原始查询
            },
            "functions": [
                {
                    // 过滤条件，只有符合条件的文档，才会被重新计算score
                    "filters": {
                        "term": {"id": "1"}
                    },
                    // 算分函数，算分函数的结果称为function score，会与query score运算，得到新算分
                    "weight": 10
                }
            ],
            // 加权模式，定义了functions score与query score的运算方式
            "boost_mode": "multiply"
        }
    }
}
```

## bool
布尔查询必须是一个或多个查询子句的组合。组合的方式有
1. must：必须匹配每个子查询，逻辑与
2. should：匹配任意一个子查询，逻辑或
3. must_not：必须不匹配，该子查询不参与算分，逻辑非
4. filter：必须匹配，但不参与算分
```json
{
    "query": {
        "bool": {
            "must": [
                // 若干子查询
            ],
            "should": [
                // 若干子查询
            ],
            "must_not": [
                // 若干子查询
            ],
            "filter": [
                // 若干子查询
            ],
        }
    }
}
```

## 操作符
在range查询中使用的操作符
1. gte：大于等于
2. lte：小于等于

## 相关性算分
文档搜索结果字段`_score`存储了文档结果与搜索词条的相关性，5.0版本前使用TF-IDF算法，5.1以后版本使用BM25算法

### TF-IDF
$$TF=\frac{词条出现次数}{文档中词条总数}$$
$$IDF=\lg(\frac{文档总数}{包含词条的文档总数})$$
$$score=\sum_{i}^n (TF * IDF)$$
1. TF：词条频率
2. IDF：逆文档频率
3. i、n：词条数

### BM25
BM25算法的得分随着词频的增加，得分增长数越低
$$Sorce(Q,D)=\sum_{i}^n \lg(1+\frac{N-n+0.5}{n+0.5}) * \frac{f_i}{f_i+k_1*(1-b+b*\frac{dl}{avgdl})}$$

## 算分函数
用于function_score查询，返回一个值，用该值来修改查询算分
1. weight：返回值为常量
2. field_value_factor：使用某个字段值作为函数结果
3. random_score：返回一个随机数
4. script_score：自定义计算公式，公式结果作为返回值

## 加权方式
1. multiply：function score乘以query score，默认值
2. replace：用function score替换query score
3. 其他：sum、avg、max、min

## 排序
es支持对搜索结果进行排序，默认是根据`_score`进行排序。可排序的类型有keyword、数值、地理坐标位置、日期类型等。**指定排序字段后，将不会对搜索结果进行打分**
```json
{
    "query": {
        "match_all": {
            }
    },
    "sort": [
        // 可以根据多个字段进行排序
        {
            // 简单类型排序
            // asc正序，desc倒序
            "字段名": "desc"
        },
        {
            // 地址位置排序
            "_geo_distance": {
                "字段名": "当前点经纬度",
                "order": "asc",
                "unit": "km"  // 结果距离的单位
            }
        }
    ]
}
```

## 分页
es对返回结果默认进行分页，每页默认10条，默认返回第一页
```json
{
    "query": {
        "match_all": {
            }
    },
    "from": 990, // 偏移量，默认为0
    "size": 10,  // 期望获取的文档数
}
```

## 深度分页解决方案
为了避免深分页造成过高内存占用，es不允许`from + size`的值大于10000。如果需要进行深度分页，可以采用以下解决方案
1. search after：分页时需要排序，原理是从上一次的排序值开始，查询下一页的数据。官方推荐方式
   1. 缺点：只能向后逐页查询，不支持随机翻页
2. scroll：原理将排序数据生成快照，保存在内存。官方已不推荐使用

## 关键词高亮
es可以给关键字添加标签，进行高亮展示。
```json
{
    "query": {
        // 如果要进行高亮，必须进行非全匹配查询
        "match": {
            "字段名": "查询内容"
        }
    },
    "highlight": {
        "fields": { // 指定要高亮的字段，可以指定多个
            "字段名": {
                "pre_tags": "<em>",    // 默认值
                "after_tags": "</em>", // 默认值
                "require_field_match": true  // 默认值true，表示高亮的字段需要与搜索字段一致
            }
        }
    }
}
```