# 数据操作语言DML

1. #### 插入
   
   语法1：
   
   ```sql
   insert into 表名(列名，列名，...)
   values(值，值， ...);
   ```
   
   > 1. 插入值的类型要和列的类型一致或兼容。
   > 
   > 2. 列和值得个数必须一致。
   > 
   > 3. nullable或者有默认值的字段可以不写。
   > 
   > 4. 可以省略列名，插入值得顺序必须和表中一致。
   
   语法2：
   
   ```sql
   insert into 表名
   set 列名=值， 列名=值...;
   ```
   
   两种语法的区别：
   
   1. 语法1支持一次性插入多行，语法2不支持。
   
   2. 语法1支持子查询，语法2不支持。
      
      ```sql
      insert into 表(列，列...)
      select from 表;
      ```

2. #### 修改
   
   1. 修改单表记录
      
      ```sql
      update 表名
      set 列=新值, 列=新值
      where 筛选条件;
      ```
   
   2. 修改多表记录（级联） 
      
      ```sql
      // 99语法
      update 表1 别名 
      inner|left|right join 表2 别名
      on 连接条件
      set 列=新值;
      ```
      
      ```sql
      // 修改张无忌女朋友的手机号为114
      update boys bo 
      inner join beauty b
      on bo.id = b.boyfriend_id
      set b.phone = '114'
      where bo.boyName = '张无忌';
      ```
      
      ```sql
      // 修改没有男朋友的女神的男朋友编号为2
      update beauty b
      left join boys bo 
      on bo.id = b.boyfriend_id 
      set b.boyfriend_id = 2
      where bo.id is null;
      ```

3. #### 删除
   
   1. delete方式删除
      
      1. 单表删除
         
         ```sql
         delete [表名] # 如果只from一张表可以省略
         from 表名
         where 筛选条件;
         ```
      
      2. 多表删除
         
         ```sql
         // sql99语法
         delete 要删除记录的表的别名
         from 表1 别名 [inner|left|right] join 表2 别名
         on 连接条件
         where 筛选条件;
         ```
         
         ```sql
         // 删除张无忌的女朋友信息
         delete b, bo
         from boys bo inner join beauty b 
         on bo.id = b.boyfriend_id 
         where bo.boyName = '张无忌';
         ```
   
   2. truncate 删除表中所有数据
      
      ```sql
      truncate table 表名; // 删除整个表的数据
      ```
      
      > 区别：
      > 
      > 1. delete不会重置自增长字段，truncate会重置自增长字段。
      > 
      > 2. truncate不能使用where条件。
      > 
      > 3. truncate删除没有返回值，delete删除有返回值。 
      > 
      > 4. truncate删除不能回滚，delete可以回滚。
