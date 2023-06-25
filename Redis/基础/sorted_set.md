# sorted_set

## 简介
- 新的存储需求：数据排序有利于数据的展示，需要提供一种可根据自身特征进行排序的方式
- 需要的存储结构：可以保存可以排序的数据结构
- sorted_set：在set基础上添加可排序的字段score

## 基础操作
- 添加数据  
  `zadd key score1 member1 score2 member2 ...`  
- 查询数据，也可以将score一并返回，这里使用的条件是索引  
  `zrange key start stop [withscores]`  
  `zrevrange key start stop [withscores]`  
- 删除数据  
  `zrem key member1 ...`
- 按条件获取数据，这里使用的条件是score字段  
  `zrangebyscore key min max [withscores] [limit]`  
  `zrevrangebyscore key max min [withscores] [limit]`  
- 按条件删除数据  
  `zremrangebyrank key start stop`  
  `zremfangebyscore key min max`  
- 获取集合数据总量  
  `zcard key`  
  `zcount key min max`  
- 集合交集，将指定数量的集合求交集放入目标key中，score默认会被求和  
  `zintersotre destination numkeys key1 key2 ... [aggreagre sum|min|max]`

## 扩展操作
- 获得数据对应的索引  
  `zrank key member`
  `zrevrank key member`
- score值的获得与修改  
  `score key member`  
  `zincrby key increment member`  

## 注意事项
- score保存的数据空间是64位
- score保存的数据可以是双精度浮点型，基于双精度浮点型的数据，可能会丢失精度
- sorted_set底层是基于set的，因此数据不能重复，如果添加重复的数据，score值将会被覆盖，保留最后一次修改的结果


