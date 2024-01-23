# Complete
elasitcsearch提供了Completion Suggester查询来实现相关的自动补全，该查询会匹配以用户输入内容开头的词条并返回

## 条件
1. 参与补全的字段必须是completion类型
2. 字段的内容一般是用来补全的多个词条(在插入时直接处理好)形成的数组，**这是由于自动补全功能只会从词条的开始进行补全**

## 语法
```json
GET /<index_name>/_search
{
    "suggest": {
        "补全查询的名称": {
            "text": "输入内容",
            "completion": {     // 自动补全的类型
                "field": "补全查询的字段",  // 字段必须是completion类型
                "skip_duplicates": true,  // 跳过重复
                "size": 10   // 获取条目数
            }
        }
    }
}
```

## 自动补全类型
1. completion
2. term