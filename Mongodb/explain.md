# explain

1. 定义
   
   ```js
   db.collection.explain().<method(...)>
   ```
   
   使用explain能分析以下命令：
   
   1. aggregate()
   
   2. count()
   
   3. distinct()
   
   4. find()
   
   5. group()
   
   6. remove()
   
   7. update()

2. 返回值解析
   
   1. 返回值（没有使用到索引）
      
      ```js
      {
              "queryPlanner" : {
                      "plannerVersion" : 1,
                      "namespace" : "test.total_statistics",
                      "indexFilterSet" : false,
                      "parsedQuery" : {
                              "year" : {
                                      "$eq" : 2020
                              }
                      },
                      "winningPlan" : {
                              "stage" : "COLLSCAN",
                              "filter" : {
                                      "year" : {
                                              "$eq" : 2020
                                      }
                              },
                              "direction" : "forward"
                      },
                      "rejectedPlans" : [ ]
              },
              "serverInfo" : {
                      "host" : "iZ8vbir71f4i102ruz9qehZ",
                      "port" : 27017,
                      "version" : "4.0.16",
                      "gitVersion" : "2a5433168a53044cb6b4fa8083e4cfd7ba142221"
              },
              "ok" : 1
      }
      ```
   
   2. 使用到索引
      
      ```js
      {
              "queryPlanner" : {
                      "plannerVersion" : 1,
                      "namespace" : "test.total_statistics",
                      "indexFilterSet" : false,
                      "parsedQuery" : {
                              "bot" : {
                                      "$eq" : "xxxxxxxxx"
                              }
                      },
                      "winningPlan" : {
                              "stage" : "FETCH",
                              "inputStage" : {
                                      "stage" : "IXSCAN",
                                      "keyPattern" : {
                                              "bot" : 1
                                      },
                                      "indexName" : "bot_1",
                                      "isMultiKey" : false,
                                      "multiKeyPaths" : {
                                              "bot" : [ ]
                                      },
                                      "isUnique" : false,
                                      "isSparse" : false,
                                      "isPartial" : false,
                                      "indexVersion" : 2,
                                      "direction" : "forward",
                                      "indexBounds" : {
                                              "bot" : [
                                                      "[\"xxxxxxxxx\", \"xxxxxxxxx\"]"
                                              ]
                                      }
                              }
                      },
                      "rejectedPlans" : [ ]
              },
              "serverInfo" : {
                      "host" : "iZ8vbir71f4i102ruz9qehZ",
                      "port" : 27017,
                      "version" : "4.0.16",
                      "gitVersion" : "2a5433168a53044cb6b4fa8083e4cfd7ba142221"
              },
              "ok" : 1
      }
      ```
   
   3. 分析
      
      分析的重点在`queryPlanner`字段，这个字段包含着查询命令的执行细节。
      
      | 字段          | 值   | 说明                       |
      | ----------- | --- | ------------------------ |
      | winningPlan | obj | 数据库在几种方法中选出来的它认为最有效的查询方法 |
      |             |     |                          |
      |             |     |                          |
      
      1. `winningPlan`字段解析
         
         | 字段         | 值                  | 说明                    |
         | ---------- | ------------------ | --------------------- |
         | stage      | COLLSCAN           | 全文档扫描（没有使用到索引）        |
         | stage      | IXSCAN             | 索引扫描                  |
         | stage      | FETCH              | 根据索引中指向的文档地址取出文档      |
         | stage      | PROJECTION         | 投影的字段已经包含在索引中，不用再读取文档 |
         | stage      | SORT               | 内存排序                  |
         | stage      | SORT_KEY_GENERATOR |                       |
         | keyPattern | obj                | 使用到的索引的定义             |
         | indexName  | str                | 使用到的索引名               |
         | isMultiKey | boolean            | 索引是否是多键索引             |
         |            |                    |                       |
