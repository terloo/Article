# 锁

## 基础操作
- 对key添加监视锁，再执行exec前如果key发生了变化，终止事务执行  
  `watch  key1 key2 ...`  
- 取消对所有key的监视  
  `unwatch`

## 注意事项
- `watch`和`unwatch`只能再`multi`之前执行


## 分布式锁
- 使用`setnx`设置一个公共锁  
  `setnx lock-key value`  

  > 利用setnx命令的返回值特征，有值则返回设置失败，无值则返回设置成功。
  > 对于返回设置成功的，拥有控制权，进行下一步的具体业务操作
  > 对于返回设置失败的，不具有控制权，排队或者返回
- 使用`del key`来释放锁
- 分布式锁是一个设计层面的概念，与redis本身的功能无关
  
## 死锁
- 如果某个客户端再设置了分布式锁之后因为某种原因（宕机）设置了分布式锁而没有释放，则产生死锁。所以必须在系统层面给锁一个限制。
- 为锁key添加时间限定，到时不释放则自动删除  
  `expire lock-key seconds`  
  `expire lock-key milliseconds`