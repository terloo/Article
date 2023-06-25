# Mongodb分片集群

1. ### 概述
   
   mongodb的副本集可以解决数据备份、读取性能的问题。但由于副本集的每个mongod数据是一模一样的，所以无法解决存储数据量过大的问题。
   
   mongodb分片技术能把数据分为两份存储，假如一个数据库中有一亿条数据，使用分片技术能将这一亿条数据分为两份分别存储在不同的数据库中。
   
   分片中的每个mongod都需要使用副本集技术防止数据丢失。

2. ### mongodb的分片集群中的mongo分为三种角色
   
   - router角色
     
      路由服务，提供入口，使得分片对外透明。router不存储数据。
   
   - configsvr角色
     
     配置服务，存储元数据信息。分片集群后端有多份存储，读取数据应该去哪读依赖于此角色，建议使用副本集。
   
   - shardsvr角色
     
     存储服务，不少于两个，存储真正的数据，建议使用副本集。

3. ### 依赖关系
   
   当用户通过router插入数据时，需要从configsvr知道这份数据插入到哪个节点，然后执行插入动作将数据插入到shardsvr。
   
   当用户通过router查询数据时，需要从configsvr知道这份数据是存储到哪的，然后再去对应的shardsvr读取数据。

4. ### mongodb分配集群的搭建
   
   1. 搭建说明：
      
      使用同一份mongodb的二进制文件
      
      修改对应的配置就能实现分片集群的搭建
   
   2. 架构设计：
      
      configsvr：使用28017，28018，28019三个端口来搭建
      
      router：使用27017，27018，27019三个端口来搭建
      
      shardsvr： 使用29017，29018，29019，29020四个端口来搭建
      
      （分为两个片，每个分片拥有一个主库一个副库的副本集）
   
   3. ##### mongodb分片configsvr角色的搭建
      
      ```yaml
      # 数据库配置文件
      systemLog:
       destination: file
       logAppend: true
       path: /var/log/mongodb/sharding/configsvr/28017/mongo.log
      storage:
       dbPath: /data/mongodb/sharding/configsvr/28017/
       journal:
        enabled: true
      processManagement:
       fork: true
      net:
       port: 28017
       bindIp: 0.0.0.0
      replication:
       replSetName: configsvr-mongo
      sharding:
       clusterRole: configsvr
      ```
      
      ```json
      // 副本集配置信息
      config = {
         _id:"configsvr-mongo",
         configsvr:true,   //  重要！！
         members:[
             {
                 _id:0,
                 host:"127.0.0.1:28017"
             },
             {
                 _id:1,
                 host:"127.0.0.1:28018"
             },
             {
                 _id:2,
                 host:"127.0.0.1:28019"
             }
         ]
      }
      ```
   
   4. ##### mongodb分片router角色的搭建
      
      ```yaml
      systemLog:
       destination: file
       logAppend: true
       path: /var/log/mongodb/sharding/router/27017/mongo.log
      # 不需要存储
      processManagement:
       fork: ture
      net: 
       port: 27017
       bindIp: 0.0.0.0
      # 没有存储数据，所以不需要副本集，但是可以搭建多个，要要配置好configDB
      sharding:
      # 指定配置角色所在的位置   重要！！！
      # 配置角色副本集名-配置角色的ip端口
       configDB: config-mongo/127.0.0.1:28017,127.0.0.1:28018,127.0.0.1:28019
      ```
      
      配置好后使用`mongos`进行启动，不用配置副本集
   
   5. ##### mongodb分片shardsvr的搭建
      
      ```yaml
      systemLog:
       destination: file
       logAppend: true
       path: /var/log/mongodb/sharding/shardsvr/29017/mongo.log
      storage:
       dbPath: /data/mongodb/sharding/shardsvr/29017/
       journal: 
        enabled: true
      processManagement:
       fork: true
      net:
       port: 29017
       bindIp: 0.0.0.0
      replication:
       replSetName: shardsvr-mongo1
      sharding:
       clusRole: shardsvr
      ```
      
      副本集配置信息为一般副本集配置信息
   
   6. ##### 向router中添加shardsvr的信息
      
      ```javascript
      sh.addShard("shardsvr-mongo1/127.0.0.1:29017,127.0.0.1:29018")
      sh.addShard("shardsvr-mongo2/127.0.0.1:29019,127.0.0.1:29020")
      sh.status()
      ```
   
   7. ##### 在router中配置数据分发规则
      
      针对某个数据库使用hash分片存储，分片存储会使同一个collection分配到多个shardsvr角色
      
      ```javascript
      use admin
      db.runCommand({enablesharding: "<数据库名称>"})
      db.runCommand({shardcollection: "<数据库名.集合名>", key:{_id::"hashed"}})
      ```
