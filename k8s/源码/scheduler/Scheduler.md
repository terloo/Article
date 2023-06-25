# Scheduler
默认调度器的主要职责，就是为新创建出来的Pod，寻找一个合适的节点(Node)  

## 最合适
1. 从集群中所有节点，根据调度算法挑选出可以运行该Pod的节点
2. 从1的结果中，选择一个最符合条件的节点
在具体的调度中，默认调度器会选择一组叫做Predicate的算法来检查每个Node。  
然后对Predicate算法中通过的Node再调用Priority的调度算法给Node打分，分最高的Node，即是调度结果

## Scheduling Framework
Scheduler为了更好的扩展性以及维护性，经过迭代重构，成为了调度框架(Scheduling Framework)。  
Scheduling Framework淡化了以前的`Predicate & Priority`概念和webhook方式的扩展机制`Scheduler Extender`。  
取而代之的是`Plugin & Extension Points`，即通过在各个预定义的扩展点，插入Plugin的方法扩展Scheduler的功能。  
核心的调度算法也放到了Plugin中，如果调度器默认的Plugin无法满足需求，可以自己实现调度算法。

## 优先级队列和缓存
1. 优先级队列保障Pod的调度顺序，结合Plugin机制，用户也可以自定义优先级算法
2. 缓存(Informer)优化调度性能，主要缓存Pod和Node的信息

## Informer
Scheduler的Informer是一切事件来源，监听了很多的对象状态，主要是Pod和Node。Pod的事件分为两种
1. 当没有调度成功的pod的增删改事件时，该pod就会被放入到优先级队列中
2. 当已经调度成功的pod发生增删改事件时，该pod就会被对应的更新到缓存中
3. node的增删改事件也会将信息更新到缓存中

## PriorityQueue
PriorityQueue由三个队列组成：
1. activeQ：用来存放将要进行调度的pod，底层数据结构是用`堆`来实现的优先级队列
2. unschedulableQ：用来存放调度失败的pod，底层数据结构是map
3. podBackoffQ：用来作为activeQ和unschedulableQ之间的一个缓存队列，也是一个用`堆`实现的优先级队列

### PodPriority
k8s可以给Pod配置一个PodPriority字段作为权重值，该值越高，在优先级队列中的排序就越前。此处的队列就是activeQ。  
activeQ的排序算法可以通过FrameWork插件机制来指定，默认的排序插件就是按照pod的PodPriority进行排序，如果没有指定，则按照加入队列的时间戳来排序，也即退化为一个FIFO队列。

### 消费者
消费者从activeQ中取出一个pod进行调度，如果调度失败。  
则将pod放入unschedulableQ中，而unschedulableQ中的pod又会周期性的被移到activeQ或者podBackoffQ中，等待重新调度

### podBackoffQ
podBackoffQ的存在，只要是为了解决优先级调度算法中存在的`无穷阻塞`或者是`饥饿`问题。即高优先级的pod总是被优先调度而低优先级pod一直得不到调度的问题。
podBackoffQ也是优先级队列，不过它的排序算法不是按照pod的权重，而是按照pod加入到unschedulableQ中的时间来决定的。  
Scheduler会周期性的将超过了podBackoffQ时间的Pod，从podBackoffQ移动到activeQ，进行再次调度。

## Cache
在Informer缓存的基础上，再进行缓存如node上所有pod的request之和，或者该node分配出去的端口号等数据。  
为了应对node和pod频繁变化造成的调度一致性问题，在每次调度pod时，都会将缓存做一次snapshot，保证了调度一致性。

## 扩展点
Scheduler Framework分为两部分：
1. Scheduling Cycle(为pod寻找最合适的node)：PreFilter、Filter、PreScore、Score、Reserve、Unreserve、Permit
    1. Filter用于过滤符合调度条件的node
    2. Score用于给符合条件的node打分
    3. Reserve用于为该pod保留资源，如volume等
    4. Permit类似于准入规则，只有符合该规则的pod，才会进行Bind阶段
2. Binding Cycle(将pod和node进行绑定的过程)：PreBind、Bind、PostBind
   1. Bind将pod的绑定关系写入到数据库，异步执行
3. assume用于在调度完成后提前更新缓存，防止Informer缓存更新不及时造成的调度不一致