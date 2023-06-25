# WorkQueue
k8s工作队列，与一般的FIFO队列不同，具有很多新特性

1. Add：添加元素
2. Len：队列中所有元素个数
3. Get：从队列中获取一个元素
4. Done：标记一个元素是否被处理
5. ShutDown：关闭队列
6. ShutDownWithDrain：流水型关闭队列
7. ShuttingDown：标记队列是否正在关闭

## 特性
1. 有序：按添加顺序处理元素(item)
2. 去重：相同元素在同一时间不会被重复处理，例如一个元素在处理之前被添加了很多次，它只会被处理一次
3. 并发性：多生产者和消费者
4. 标记机制：支持标记机制，标记一个元素是否被处理，也允许元素在处理时重新排队
5. 通知机制：ShutDown方法通过信号量通知队列不再接受新的元素，并通知metric goroutine退出
6. 延迟：支持延迟队列，延迟一段时间后再将元素存入队列
7. 限速：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队的次数(Requeued)
8. Metric：支持metric监控指标，可用于prometheus监控

## 3种队列
1. Interface：FIFO队列，先进先出，并支持去重机制
2. DelayingInterface：延迟队列，基于Interface进行封装。
3. RateLimitingInterface：限速接口，基于DelayingInterface进行封装，支持元素存入队列时进行速率限制。