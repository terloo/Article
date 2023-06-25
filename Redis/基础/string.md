# string
## 简介
- 存储的数据：单个数据，最简单的数据存储类型，也是最常用的数据存储类型
- 存储数据的格式：一个存储空间保存一个数据
- 存储内容：通常使用字符串，如果字符串以整数的形式展示，可以作为数字操作
## 基本操作
- 添加、修改数据  
  `set key value`
- 获取数据  
  `get key`
- 删除数据  
  `del key`
- 添加、修改多个数据  
  `mset key1 value1 key2 value2 ...`
- 获取多个数据  
  `mget key1 key2 ...`
- 获取数据字符个数（字符串长度）  
  `strlen key`
- 追加，如果key不存在则新建，返回追加后的长度  
  `append key value`
## 扩展操作
- 设置数值字符串数据增加指定范围的值，返回修改后的值  
  `incr key`  
  `incrby key increment`  
  `incrbyfloat increment`
- 设置数值字符串数据减小指定范围的值，返回修改后的值   
  `decr key`  
  `decrby key increment`  
  没有`decrbyfloat increment`命令
- 设置key有效时长  
  `setex key seconds value`  
  `psetex key milliseconds value`
## 注意事项
- 数据操作不成功的反馈与数据正常操作之间的差异
  1. 表示运行结果是否
     - (interger) 0 -> false 失败
     - (interger) 1 -> true 成功
  2. 表示运行结果
     - (interger) 3 -> 3 3个
     - (interger) 1 -> 1 1个
- 数据未获取到  
  (nil)等同于null
- 数据最大存储量  
  512M
- 数据最大计算范围(long int)  
  32Byte
