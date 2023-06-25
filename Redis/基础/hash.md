# hash
## 简介
- 对一系列存储的数据进型编组方便管理
- 一个存储空间保存多个键值对数据
  > 如果field数量比较少，存储结构优化为类数组结构
  > 如果比较多，则使用hashMap结构

## 基本操作
- 添加/修改数据  
  `hset key field value`
- 获取数据  
  `hget key field`  
  `hgetall key`
- 删除数据  
  `hdel key field1 field2 ...`
- 添加/修改多个数据  
  `hmset key field1 value1 field1 value2 ...`
- 获取多个数据  
  `hmget key field1 field2`
- 获取hash中字段数量  
  `hlen key`
- 判断hash中字段是否存在  
  `hexists key field`

## 扩展操作
- 只获取hash表中所有的字段名或字段值  
  `hkeys key`  
  `hvals key`
- 设定指定字段的数值数据增加指定范围的值  
  `hincrby key field increment`  
  `hincrbyfloat key field increment`
- 如果key中对应的fied有值则不做操作，没值则新建值
  `hsetnx key field value`

## 注意事项
- hash类型下的value值只能存储字符串，不允许其他类型或者嵌套hash。如果数据未获取到，对应的值为(nil)
- 每个hash可以存储2^32 - 1个键值对
- hash的类型十分贴近对象的数据存储形式，但是hash设计初衷不是为了存储对象，更不可将hash作为对象列表使用
- hgetall操作，如果内部field操作过多，遍历整体数据效率会很低，可能会成为数据访问瓶颈

