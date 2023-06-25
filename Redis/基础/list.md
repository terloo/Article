# list

## 简介
- 数据存储需求：存储多个数据，并对数据存入存储空间的顺序进行区分
- 需要的存储结构：一个存储空间保存多个数据，且通过数据可以体现进入顺序
- list类型：保存多个数据，底层使用双向链表的结构实现

## 基础操作
- 添加数据  
  `lpush key value1 value2 ...`  
  `rpush key value1 value2 ...`
- 获取数据  
  `lrange key start stop`  
  `lindex key index`  
  `llen key`
- 获取并移除数据  
  `lpop key`  
  `rpop key`

## 扩展操作
- 弹出数据，如果列表为空则等待指定时间，可以同时等待多个列表知道其中一个列表能弹出数据  
  `blpop key1 key2 ... timeout`  
  `brpop key1 key2 ... timeout`
- 移除指定数据，最多移除count个  
  `lrem key count value`  

## 注意事项
- list中保存的数据都是string类型的，数据总容量是有限的，最多2^32-1个元素
- list具有索引的概念，但是操作数据时一般以队列的形式进行入队出队操作，或者以栈的形式进行出栈入栈的操作。
- 获取全部数据操作，可以将lrange的结束索引设为-1
- list可以对数据进行分页操作，通常第一页的的信息来自于list，第二页以及更多的数据通过数据库进行加载