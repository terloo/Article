# CacherStorage
CacherStorage缓存层为UnderlyingStorage提供缓存，目的是快速响应并减少Etcd压力。  
UnderlyingStorage及其装饰缓存CacherStrage共同构成了apiserver的数据访问层(DAO)

## 分层设计
1. cacheWatcher：Watcher观察者管理
2. watchCache：通过Reflector框架与Underlyingstorage底层存储对象进行交互，UnderlyingStorage与Etcd进行交互并将回调事件分别存储至w.onEvent、w.cache、cache.Store中
3. Cacher：用于分发给目前所有已连接的观察者，分发过程通过非阻塞(Non-Blocking)机制实现

## 特点
1. Create、Delete、Update不经过缓存，直接与UnderlyingStorage进行交互
2. 只有Get、GetToList、List、GuaranteedUpdate、Watch、WatchList是基于缓存进行设计的
3. 其中Watch操作的事件缓存机制使用缓存滑动窗口来保证历史事件不会丢失。

## 结构体