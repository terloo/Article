# lookup聚合阶段

1. #### 定义
   
   ```js
   $lookup: {
       from: <collection to join>,
       localField: <field from the input documents>,
       foreignField: <field from the document of the "from" collection>,
       as: <output array field>
   }
   ```
   
   > from：提供同一个数据库中另一个集合的名字
   > 
   > localField：管道文档中用来进行查询的字段，如果是数组字段将会一一匹配数组中所有字段，可以使用unwind先将其展开。
   > 
   > foreign：from集合中的查询字段
   > 
   > as：写入管道文档中的查询结果数组字段名（数组中的元素为匹配到的文档）
   > 
   > 没有匹配到文档会被写入空数组

2. #### 用法
   
   1. 针对单一字段值进行查询
   
   2. 使用复杂条件进行查询
      
      ```js
      $lookup:{
          from: <collection to join>,
          let: { <var_1>:<expresion>,...,<var_n>:<expression> },
          pipeline: [ <pipeline to execute on the collection to jion> ],
          as: <output array field>
      }
      ```
      
      > pipeline: 另外一套聚合管道阶段对查询集合文档进行处理
      > 
      > let：对查询集合中的文档进行聚合操作时，如果需要使用原管道文档中的字段，则必须使用let参数对字段进行声明，可选参数。声明的变量视作系统变量，使用`$$`来引用
