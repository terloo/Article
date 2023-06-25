# MVCC
多版本并发控制，mysql读写情况的处理

## 原理
1. 在RC和RR隔离级别下的事务，执行update语句其实是向表中新插入了一行数据。
2. 聚簇索引上除了用户定义的字段，还会有mysql添加的trx_id和roll_ptr字段，在此行数据被插入时，分别存储建立这行数据的事务id和这行数据的上一版本数据行地址。形成链表的数据结构。
3. 在RC和RR隔离级别下的事务，执行普通的读取语句时，会生成一个read-view的数据结构，存储([当前所有未提交数据的事务id], 下一个事务id号)
   1. 当前事务id称为`crt_trx_id`
   2. 当前所有未提交数据的事务id称为`uncommit_trx_ids`
   3. 当前所有未提交数据的事务id中最小的事务id号称为`min_trx_id`
   4. 当前所有未提交数据的事务id中最大事务id号称为`max_trx_id`
   5. 下一个事务id号称为`next_trx_id`
4. select在读取数据时，会从头遍历链表依次进行以下判断。通过返回值来确认是否可见。
   1. 如果该行数据的`trx_id < min_trx_id`，即该行数据是由以前某个已提交的事务更新的，返回true
   2. 如果该行数据的`trx_id >= next_trx_id`，即该行数据是由比此事务更新的事务更新的，返回false
   3. 如果该行数据的`trx_id in crt_trx_id`，即该行数据是此事务产生时的未提交事务更新的，返回false
   4. 返回true
5. RC隔离级别每次普通select都会生成一个read-view，而RR隔离级别再第一次select生成read-view，此后一直使用该read-view。
