# Elasticsearch
Elasticsearch是一个开源的大文本数据库和搜索工具

## 概念
| 概念名 | 英文     | 说明                                     | 对应Mysql的概念 |
| ------ | -------- | ---------------------------------------- | --------------- |
| 文档   | Document | 一条条数据，文档使用JSON格式存储         | 记录            |
| 索引   | Index    | 文档的集合                               | 表              |
| 字段   | Field    | JSON文档的字段                           | 列              |
| 映射   | Mapping  | 索引中文档字段的约束，例如字段类型的约束 | 表结构          |
| DSL    | DSL      | es使用的JSON风格查询语句，用于操作es     | SQL             |

## mapping属性
mapping是对索引库中文档的约束，常见的mapping属性
1. type：字段数据类型
   1. 字符串
      1. text：文本字符串，可分词的文本
      2. keyword：精确值，例如国家、品牌、ip地址等不可分词的文本
   2. 数值：long、integer、short、byte、double、float
   3. 布尔：boolean
   4. 日期：date
   5. 对象：object，对象可以任意嵌套，对象也具有子属性
   6. 地理
      1. geo_point：由经纬度确定的一个点，格式`维度, 经度`
      2. geo_shape：由多个geo_point组成的一个复杂图形
   7. 多值：es中没有数组的概念，但类似数组的，es允许一个字段拥有多个相同类型的值
2. index：是否对该字段创建索引，默认为true
3. analyzer：使用哪种分词器，用于text类型
4. properties：该字段的子字段

