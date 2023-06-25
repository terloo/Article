# Mongodb单列和副本集

1. #### 启动服务端：
   
   1. 编写配置文件：
      
      ```yml
      # 普通的单列mongo设计
      # 系统日志配置
      systemLog:
         destination: file
         logAppend: true
         path: /data/mongodb/single/mongodb.log
      # 数据储存相关配置
      storage:
          dbPath: /data/mongodb/single/
          journal: 
              enable: true
      processManagement:
          fork: true
      net:
          port:27017
          bindIp: 0.0.0.0
      ```
      
      | 参数                | 值    | 说明      |
      | ----------------- | ---- | ------- |
      | systemLog         |      | 日志相关    |
      | destination       | file | 日志输出地   |
      | path              | path | 文件路径    |
      | storage           |      | 数据库文件相关 |
      | dbpath            | path | 数据库文件路径 |
      | journal           |      | 预防数据丢失  |
      | processManagement |      | 进程管理    |
      | fork              | bool | 在后台启动   |
      | net               |      | 网络相关    |
      | port              | int  | 端口      |
      | bindIp            | IP   | 监听的ip地址 |
   
   2. 启动MongoDB
      
      mongod    -f     <配置文件路径>

2. #### 操作客户端：
   
   1. 基础概念：
      
      - database        数据库
      
      - collection       集合
      
      - filed                字段
      
      - document      文档
   
   2. 基础操作：
      
      - 查询所有的库:
        
        show dbs;
      
      - (创建并)切换库：
        
        use <库名>
      
      - 关闭mongo服务(必须进入admin数据库)：
        
        use admin;
        
        db.shutDownServer();
      
      - 创建集合：
        
        db.<集合名>.insert({'xxxxx':'xxxxx'})
        
        在向结合插入数据时会自动创建集合
      
      - 删除数据库：
        
        db.dropDatabase();
   
   3. 增删改查(基础)：
      
      1. 增：
         
         db.<集合名>.insert()
      
      2. 删：
         
         db.<集合名>.remove({xxxx:'xxxx'})    # 有条件的删除
         
         db.<集合名>.remove({})     # 删除集合所有文档，会保留集合
         
         db.<集合名>.drop()    # 删除集合，如果数据库中没有集合则自动删除
      
      3. 改：
         
         db.<集合名>.update(<条件字典>, {$set:<字典>})
      
      4. 查：
         
         db.<集合名>.find({xxxx:'xxxx'})

3. #### 查询语句：
   
   1. 易读的方式输出
      
      db.xxx.find().pretty()
   
   2. 限制输出的文档数
      
      db.xxx.find().limit(n)
   
   3. 跳过前n条文档
      
      db.xxx.find().skip(n)
   
   4. mongo的分页查询
      
      db.xxx.find().skip(0).limit(2)
      
      db.xxx.find().skip(2).limit(4)
      
      db.xxx.find().skip(4).limit(6)
      
      db.xxx.find().skip(6).limit(8)
   
   5. 排序
      
      db.xxx.find().sort({age:1})   # age升序
      
      db.xxx.find().sort({age:-1})   # age降序
   
   6. 比较查询（用于int类）
      
      db.xxx.find({age: {$lt: 30})
      
      $gt  大于
      
      $lt   小于
      
      $gte   大于等于
      
      $lte    小于等于
   
   7. 逻辑查询
      
      db.xxx.find({$or:[{name:"csj1"}, {name:"csj2"}]})
      
      db.xxx.find({$and:[{name:"csj1"}, {age:20}]})
   
   8. 正则查询
      
      db.xxx.find({$regex: {name:"csj[0-9]"}})    # 普通正则
      
      db.xxx.find({$regex: {name:"(zhangsan)"}})     #  分组正则

4. ### 索引查询与建立
   
   1. mongo有慢查询的概念，超过100ms的查询会被写入慢查询日志mongodb.log
      
      db.getProfilingstatus()     # 能设置时间阈值
      
      db.xxx.find({xxx}).explain(true)  # 是否输出此次查询的详细信息，查询没有索引的字段会进行全表扫描
   
   2. 建立索引
      
      1. 查看该表已有的索引
         
         db.xxx.getIndexes() 
      
      2. 添加索引
         
         db.xxx.ensureIndex({ age: 1 })   # 添加升序索引
         
         db.xxx.ensureIndex({ age: 1 }, {unique:true})    # 使索引成为唯一索引
      
      3. 删除索引
         
         db.xxx.dropIndex({ age: 1 })    # 删除索引

5. ### mongo数据库监控
   
   1. 实时监控
      
      mongostat -h 127.0.0.1
   
   2. 获取状态信息
      
      db.serverStatus()   # 获取所有状态信息
      
      db.serverStatus().network  # 单独查看网络信息
      
      db.serverStatus().opcounters  # 统计增删改查的次数
      
      db.serverStatus().connections  # 连接信息

6. ### mongodb副本集
   
   mongodb的副本集能预防数据丢失，多态mongo的数据一致
   
   mongodb的副本能在有问题时自动切换
   
   1. 配置文件
      
      ```yml
      # 集群mongodb配置文件(主数据库)
      systemLog:
       destiantion: file
       logAppend: true
       path: /data/mul-mongodb/27017/mongolog.log # 需要配置
      storage:
       dbPath: /data/mul-mongodb/27017/ # 需要配置
       journal:
        enabled: true
      net:
       port: 27017 # 需要配置
       bindIp: 0.0.0.0
      replication:
       replSetName: repl-mongo  # 副本集名称，需保持一致
      ```
   
   2. 副本集的配置
      
      启动三个mongodb服务，需要更改配置文件中的端口、数据存储路径、日志路径。
   
   3. 初始化副本集
      
      1. 进入其中一个mongo（权重相同时，将会成为主数据库），创建变量config
         
         ```json
         config = {
            _id:"repl-mongo",  // 配置文件中的副本集名   
            members:[
                {
                    _id:0,
                    host:"127.0.0.1:27017"
                },
                {
                    _id:1,
                    host:"127.0.0.1:27018"
                },
                {
                    _id:2,
                    host:"127.0.0.1:27019"
                }
            ]
         }
         ```
      
      2. 初始化副本集
         
         rs.initate( config )
         
         将会同步主数据库数据到副数据库
      
      3. 查看副本集状态
         
         rs.status()
         
         PRIMARY: 主    只有主副本能写入
         
         SECONDARY: 副
      
      4. 查询副数据库数据
         
         需要先声明rs.slaveOk()
      
      5. mongodb选择主数据库的方式依赖于权重
         
         默认权重都是1，可以通过修改config的方式来修改权重
         
         conf = rs.config()   # 先获取当前配置信息到一个变量，注意实例位置
         
         conf.members[2].priority = 3
         
         conf.members[1].priority = 2
         
         conf.members[0].priority = 1
         
         rs.initiate(conf)
         
         即使主数据库关闭，在重启后主数据库会抢夺主
      
      6. mongodb副本集的伸缩
         
         1. 往现有的集群中添加实例，集群名要保持一致
            
            use admin
            
            rs.add("127.0.0.1:27020")   #  会自动同步所有数据
            
            用rs.add 被添加进的实例权重默认为1
         
         2. 从副本集中移除实例
            
            use admin
            
            rs.remove("127.0.0.1:27017")   #  不可移除primary
         
         3. 修改过后的集群要注意修改权重，注意实例位

7. ### mongo的备份
   
   1. 备份的工具
      
      mongodump：用来备份数据
      
      mongorestore：用来导入备份的数据
   
   2. 备份的说明
      
      单台服务器直接使用mongodump指定ip和端口进行备份
      
      集群需要连接到primary进行备份
   
   3. 备份所有库
      
      mongodump -h 127.0.0.1 -p 27017 -o /data/mongo/backup/27017/
      
      注意：每一个数据库都会生成一个备份文件
   
   4. 导入备份数据
      
      mongorestore -h 127.0.0.1 -p 27018 /data/mongo/backup/27017
      
      注意：如果指定的是一个目录将会导入目录下所有的备份文件
