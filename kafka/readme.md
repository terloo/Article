# kafka

## 定义

kafka是一个分布式的基于分布/订阅模型的消息队列

## 消息队列的好处

1. 解耦
2. 可恢复性
3. 缓冲：处理生产大于消费速度的问题
4. 削峰
5. 异步通信

## 消息队列的两种模式

1. 点对点：消费者主动拉取数据，消息收到后消息清楚
2. 发布/订阅模式：一对多，消费者消费数据后不会清楚数据。kafka的订阅是消费者主动拉取消息而不是队列推送消息到消费者，所以消费者需要长轮询

## kafka架构(基础)

1. 生产者
2. 消费者组：某一个Topic只能被一个消费者组里面的某一个消费者所消费
3. kafka集群
  1. Broker：kafka服务(进程)
  2. Topic：Topic是消息的命名空间，为逻辑划分
  3. Partition：每个Topic可以划分为若干个Partition，Partition会按照算法尽可能均匀的分布到Broker。消息会随机发往不同的Partition，以此来提高集群的扩展能力。1 到 32767之间。
  4. Leader与Follower(replication-factor)：一条消息会在一个Broker中被设置为Leader，其他Broker中保存副本设置为Follower，以防止数据丢失。replication-factor数为leader+follower的数量。不能超过broker的数量。
  5. Zookeeper：存储集群节点信息
  6. 磁盘：kafka的数据存储再磁盘中，默认保存7天(168h)

## kafka架构(高级)

### 工作流程

1. 每个Partition都对应着一个文件夹，维护着从头开始的独立的当前信息偏移量。Producer生产的消息将会随机发往一个Partition。
2. Conusmor轮询到有新消息产生时，将会拉取消息。
3. Partition根据自己记录的Consumer偏移量来发送消息。
4. 由于消息是由Partitio并行发送的，所以Consumer接受的消息顺序不一定于生产消息顺序一致。

### Partition目录文件

由于生产者生产的消息会不断追加到log文件末尾，所以kafka使用了分片加索引的机制来保存文件

1. .log：消息存储文件，默认最大1g，最多存储7天。超过1g会创建新的.log文件。文件名是保存的所有消息的最小偏移量。
2. .index：索引信息，保存着某个offset消息存在于某个.log的位置信息，可以快速定位某个offset所在的.log文件的位置。
3. 当要寻找某个消息时，首先通过文件名来寻找消息的位置信息保存在哪个.index中，再在.index中索引出消息在.log文件中的对应位置，最后在对应的.log文件中读取消息

### 生产者分配分区策略

1. 指明Partition值时，直接使用指定的Partition值
2. 没有Partition但是有key的情况下，使用key的hash值模分区数的方法计算Partition值
3. 既没有Partition又没有key时，使用round-robin算法，第一条随机

### 生产者ISR

leader再收到数据后，需要同步至follower，所有follower全部同步后，才可以向生产者响应ack。  
如果有follower故障，那么leader就无法响应ack。kafka采用ISR(in-sync relica set)来避免这个情况。leader出现故障后，kafka会从isr中重新选举leader。

1. Leader维护了一个动态的in-sync replica set，意为和leader保持同步的follower集合。
2. 当ISR中的follower完成数据同步后，leader就会给生产者发送ack。
3. 如果follower长时间未向leader同步数据，则将该follower提出ISR，该时间阈值由`replica.lag.time.max.ms`参数设定。Leader发生故障后，就会从ISR中选举新的leader

### 生产者ack机制

对于某些不太重要的参数，对数据可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等ISR中所有的follower全部同步成功再响应。所以kafka提供了三种可靠性级别：

1. acks为0：producer不等待broker的ack，broker接收到数据，还未写入磁盘即返回。
2. acks为1：producer等待broker的ack，broker接收到数据，leader写入磁盘后即返回ack，不等待follower同步完成。
3. acks为-1：producer等待broker的ack，broker接收到数据，leader写入磁盘并在isr中所有follower全部同步完成后返回ack。

## 消费数据一致性

1. LEO：每个副本的最后一个offset
2. HW：所有副本中最小的一个LEO
3. follower故障：follower发生故障会被临时踢出ISR，待follower恢复后，follower会读取本地磁盘记录的上次HW，并将高于HW的所有数据截掉，从HW开始向leader请求同步。直到该follower的HW大于等于HW时，重新加入ISR。
4. leader故障：leader故障后，会从ISR中重新选取一个新leader，为保证数据一致性，follower会将各自log文件中大于HW的部分截取掉。然后从新leader中同步数据。
5. 消费者只能消费到小于等于HW的消息。

> 以上措施只能保证数据一致性，不能保证数据不丢失或者数据重复

## 幂等性

开启幂等性后，Producer在连接上kafka时会获得一个PID(producerID)，此后每次发送消息时，都会给消息一个<PID, Partition, SeqNum(序列化号)>组成的唯一id。Broker会缓存这个唯一id，如果收到相同的唯一id信息，将会直接丢弃。  
开始方式：Producer的enable.idompotence设置为true。这个设置将会将使acks值默认为-1。

## 分区分配策略

一个consumer group中有多个consumer，一个topic又有多个partition，所以必须会涉及到partition分配的问题，即确定那个partition由哪个consumer来消费。
kafka有两种分配策略：

1. RoundRobin：消费者组订阅的所有topic的所有partition会被hash之后进行排序，在轮询分配给consumer
2. Range：消费者组订阅的同一个Topic的所有partition会分别轮询分配consumer

## 消费者offset的维护

1. 消费者组+partition+topic共同维护一个offset。消费者组的消费者数量改变时，新的消费者将会按照这个offset去对应的partition消费对应offset的数据。

## kafka在zookeeper中的信息

1. /controller：存储kafka主broker信息
2. /brokers

   1. /ids：已注册的brokerid
   2. /topics：已创建的主题

3. /config：kafka的配置信息
4. /consumers：保存consumer信息，*已废弃*，放在fafka本地，使用topic保存

> 连接__consumer_offsets
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic __consumer_offsets --consumer.config config/consumer.properties --from-beginning --formatter "kafka.coordinator.group. GroupMetadataManager\$OffsetsMessageFormatter"


## 生产者事务
为了实现跨分区跨会话事务，需要引入一个全局唯一的TransactionID，并将Producer获得的PID和TransactionID唯一对应。这样即使Producer重启，也会获得相同的PID，不会产生消息重复。
