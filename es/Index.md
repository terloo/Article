# Index
若干文档的集合，被称为Index

## 创建索引库
```json
PUT /索引库名称
{
    "mappings":{
        "properties": {
            "字段名1": {
                "type": "text",
                // 创建索引时使用的分词器
                "analyzer": "ik_smart",
                // 搜索时使用的分词器
                "search_analyzer": ""
            },
            "字段名2": {
                "type": "keyword",
                "index": false,
            },
            "字段名3": {
                "type": "object",
                "properties": {
                    "子字段名1": {
                        "type": "keyword",
                        "index": false,
                    },
                    "子字段名2": {
                        "type": "integer",
                        "index": false,
                    },
                }
            },
        }
    }
}
```

## 查询索引库
```json
GET /索引库名称
```

## 删除索引库
```json
DELETE /索引库名称
```

## 修改索引库
理论上，es是禁止修改索引库的。但是，可以往已存在的索引库中添加新的字段
```json
PUT /索引库名称/_mapping
{
    "properties": {
        // 字段名必须不能重复
        "新字段名": {
            "type": "keyword"
        }
    }
}
```

## 字段拷贝
使用copy_to可以将一个字段拷贝(实际只创建了索引)至另一个字段，通过这种方式，可以实现一次索引多个字段
```json
PUT /索引库名称
{
    "mappings":{
        "properties": {
            "字段名1": {
                "type": "text",
                "analyzer": "ik_smart"
            },
            "字段名2": {
                "type": "keyword",
                "index": false,
                "copy_to": "字段名1",
            },
        }
    }
}
```