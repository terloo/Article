# Mycat

1. #### 简介
   
   mycat是一个数据库中间件

2. #### 适用场景
   
   1. 读写分离
   
   2. 数据分片
      
      1. 垂直拆分（分库）
      
      2. 水平拆分（分表）
   
   3. 多数据源整合

3. #### 安装启动
   
   1. 下载
   
   2. 解压
   
   3. 修改配置文件    在mycat/conf/中
      
      1. server.xml 定义用户以及相关系统变量，如端口等。
         
         修改`<user name='root'>`用户名避免跟mysql用户混淆
      
      2. schema.xml 定义逻辑库、表、分片等内容
         
         ```xml
         <?xml version="1.0"?>
         <!DOCTYPE mycat:schema SYSTEM "schema.dtd">
         <mycat:schema xmlns:mycat="http://io.mycat/">
                 <!-- 总览，主要是指明逻辑服务的信息和逻辑服务使用的数据节点 -->
                 <schema name="TESTDB" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1">
                 </schema>
                 <!-- 数据节点配置，配置逻辑库名，实体主机，实体数据库名 -->
                 <dataNode name="dn1" dataHost="host1" database="mysqldatabase" />
                 <!-- 实体数据存储主机配置 -->
                 <dataHost name="host1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThresHold="100">
                         <heartbeat>select user()</heartbeat> <!-- 心跳监测 -->
                         <!-- 写服务，一般配置为master服务，可以配置多个 -->
                         <writeHost host="hostM1" url="localhost:3306" user="root" password="3">
                             <!-- 读服务，一般配置为slave服务，可以配置多个 -->
                             <readHost host="hostS1" url="xxxx" user="root" password="3"></readHost>
                         </writeHost>
                 </dataHost>
         </mycat:schema>
         ```
         
         > schema【name:TESTDB】（相当于mysql服务）-> dataNode【name:dn1】（相当于mysql数据库）-> dataHost（指定读写数据用的实体服务地址）
         > 
         > 访问TESTDB
   
   4. 启动
      
      ```shell
      控制台启动
      ./mycat console
      后台启动
      ./mycat start
      ```
   
   5. 登录
      
      ```shell
      9066是mycat默认管理端口
      mysql -u用户名 -P 9066 -h xxxxx -p
      8066是mycat默认数据窗口
      mysql -u用户名 -P 8066 -h xxxxx -p
      ```

4. #### 读写分离
   
   修改schema.xml的dataHost节点的balance属性能启用读写分离功能
   
   | balance的值 | 说明                                                              |
   | --------- | --------------------------------------------------------------- |
   | 0         | 不启用读写分离机制，将所有的读写操都发送到当前可用的writeHost上。                           |
   | 1         | 全部的readHost和stand by writeHost（除了可写master以外的master）都参与读请求的负载均衡。 |
   | 2         | 所有的读操作都随机的在writeHost和readHost上进行分发。                             |
   | 3         | 所有的读请求随机的分发到readHost执行，writeHost不负担读操作。                         |

5. #### 分库
   
   将两张表分别放在两个库中，通过mycat进行读取
   
   1. 分库的基本原则
      
      但凡可能出现关联的表不能分库存储
   
   2. 分库配置
      
      ```xml
      <?xml version="1.0"?>
      <!DOCTYPE mycat:schema SYSTEM "schema.dtd">
      <mycat:schema xmlns:mycat="http://io.mycat/">
              <schema name="TESTDB" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1">
                  <!-- 单独指定某一张表在另外一个数据节点中读写 -->
                  <table name='emp' dataNode='dn2'></table>
              </schema>
              <dataNode name="dn1" dataHost="host1" database="mysqldatabase" />
              <!-- 为单独指定的表设置数据节点 -->
              <dataNode name="dn2" dataHost="host2" database="mysqldatabase2" />
              <dataHost name="host1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThresHold="100">
                      <heartbeat>select user()</heartbeat>
                      <writeHost host="hostM1" url="localhost:3306" user="root" password="3">
                          <readHost host="hostS1" url="xxxx" user="root" password="3"></readHost>
                      </writeHost>
              </dataHost>
      </mycat:schema>
      ```

6. #### 分表
   
   将一张表根据某个字段按某一规则拆开放置在两个数据库中，提高查询效率。
   
   1. 配置schema.xml
      
      ```xml
      <?xml version="1.0"?>
      <!DOCTYPE mycat:schema SYSTEM "schema.dtd">
      <mycat:schema xmlns:mycat="http://io.mycat/">
              <schema name="TESTDB" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1">
                  <table name='emp' dataNode='dn2'></table>
                  <!-- 给一张表指定两个dataNode，并指定规则 -->
                  <table name="dept" dataNode="dn1,dn2" rule="mod_rule"></table>
              </schema>
              <dataNode name="dn1" dataHost="host1" database="mysqldatabase" />
              <dataNode name="dn2" dataHost="host2" database="mysqldatabase2" />
              <dataHost name="host1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThresHold="100">
                      <heartbeat>select user()</heartbeat>
                      <writeHost host="hostM1" url="localhost:3306" user="root" password="3">
                          <readHost host="hostS1" url="xxxx" user="root" password="3"></readHost>
                      </writeHost>
              </dataHost>
      </mycat:schema>
      ```
   
   2. 在rule.xml中配置分表规则
      
      ```xml
      <mycat:rule xmlns:mycat="http://io.mycat/">
        <tableRule name="mod_rule">
                <rule>
                    <!-- 需要配置规则的列 -->
                    <columns>id</columns>
                    <!--  对列采取的规则算法，这里拿取模算法举例 -->
                    <algorithm>mod_lang</algorithm>
                </rule>
        </tableRule>
        <!-- 其余的配置不要修改 --->
        
        <!-- 函数都是由预置的java类来实现 -->
        <function name="mod-long" class="io.mycat.route.function.PartitionByMod">
                <!-- 设置节点数，列值对该值取模通过得到的返回值来分配数据存储的数据库 -->
                <property name="count">2</property>
        </function>
      </mycat:rule>
      ```
   
   3. 注意事项
      
      1. 在插入数据时，必须按标准写法指明插入数据的列和值
      
      2. 
