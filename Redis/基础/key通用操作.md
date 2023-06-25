# key通用操作

## key特征
- key是一个字符串，用过key获取redis中保存的数据

## 普通操作
- 删除指定的key  
  `del key`
- 获取key是否存在  
  `exists key`
- 获取key的类型  
  `type key`

## 时效性控制
- 为一个key设置有效期，可以使用时间戳  
  `expire key secounds`  
  `expire key milliseconds`  
  `expire key timestamp`  
  `pexpire key milliseconds-timestamp`  
- 获取key的有效期  
  `ttl key`  没有设置有效期返回-1，key不存在返回-2  
  `pttl key`  
- 切换key从时效性转为永久性  
  `persist key`  
- 查询key
  `keys pattern`

## 其他操作
- 将key改名  
  `rename key newkey`  
  `renamenx key newkey`  只有在newkey不存在时才能改名成功
- 对list, set or sorted set进行排序 默认情况下是正序，且不改变原key的value  
  `sort key`  
- 其他key通用操作  
  `help @generic`
