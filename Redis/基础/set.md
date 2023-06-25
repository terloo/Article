# set
## 简介
- 新的存储需求：存储大量的数据，在查询方面提供更高的效率
- 新的存储结构：能保存大量的数据，高效的内存存储机制，便于查询
- set类型：与hash存储结构完全相同，但仅存储键，不存储值（存为nil），并且值是不允许重复的
  
## 基本操作
- 添加数据  
  `sadd key member1 menber2`  
- 获取全部数据  
  `smembers key`
- 删除数据  
  `srem key member1 member2`  
- 获取集合数据总量  
  `scrad key`
- 判断集合中是否包含指定的数据  
  `sismember key member`

## 扩展操作
- 从集合中随机挑选指定个数的数据  
  `srandmember key [count]`
- 从集合中随机获取指定数据并将其移除  
  `spop key [count]`
- 求两个集合的交、并、差集  
  `sinter key1 key2`  
  `sunion key1 key2`  
  `sdiff key1 key2`  
- 求两个集合的交、并、差集，并将结果存储到destination中  
  `sintersotre destination key1 key2`  
  `sunion destination key1 key2`  
  `sdiff destination key1 key2`  
- 将指定数据从原始集移动到目标集合中  
  `smove source destination member`  