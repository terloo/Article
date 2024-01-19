# Aggregations
聚合查询(Aggregations query)可以实现对文档数据的统计、分析和运算

## 参与聚合的数据类型
1. keyword
2. 数值
3. 日期
4. 布尔

## 聚合分类
1. 桶(Bucket)聚合：对文档进行分组
   1. TermAggregation：按照文档字段值进行聚合，类似mysql的groupby
   2. Date Histogram：按照日期进行分组，例如一周为一组一月为一组
2. 度量(Metric)聚合：用以计算一些值，比如最大最小平均值
   1. Avg：平均值
   2. Max：最大值
   3. Min：最小值
   4. Stats：同时求max、min、avg、sum等
3. 管道(Pipeline)聚合：以其他聚合的结果为基础进行聚合

## 聚合的要素
1. 要素
   1. 聚合名称
   2. 聚合类型
   3. 聚合字段
2. 可选属性
   1. size
   2. order
   3. field
3. 子聚合aggs

## 基础返回结构
```json
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    // 普通query的返回
    "hits" : {
        "total" : {
        "value" : 500,
        "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "聚合名" : {
            // 聚合结果
            ...
        }
    }
}
```

## Bucket聚合
默认情况下，Bucket聚合会在内部统计文档数量，记为_count，如果要排序，需要按该字段名进行排序  
默认情况下，Bucket聚合是对索引库中所有文档进行聚合，可以手动限制聚合文档的范围，只需要添加query条件即可
```json
{
    "query": {    // 指定聚合范围
        ...
    },
    "size": 0,   // 设置size为0，查询结果中不返回被聚合的文档，只包含聚合结果
    "aggs": {    // 可以定义多个聚合
        "聚合名": {  // 聚合需要一个名字
            "terms": {  // 聚合的类型
                "field": "字段名",  // 参与聚合的字段
                "order": {
                    "_count": "asc"
                },
                "size": 20  // 希望获取的聚合结果数量
            }
        }
    }
}
```

## Metrics聚合
Metrics可以进行嵌套(子聚合)
```json
{
    "size": 0,   // 设置size为0，查询结果中不返回被聚合的文档，只包含聚合结果
    "aggs": {    // 可以定义多个聚合
        "聚合名": {  // 聚合需要一个名字
            "terms": {  // 聚合的类型
                "field": "字段名",  // 参与聚合的字段
                "order": {
                    "_count": "asc"
                },
                "size": 20  // 希望获取的聚合结果数量
            },
            "aggs": {  // 子聚合
                "子聚合名": {
                    "field": "字段名"
                }
            }
        }
    }
}
```