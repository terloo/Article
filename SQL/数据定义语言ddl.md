# 数据定义语言DDL

1. #### 库的管理
   
   1. 创建
      
      ```sql
      create database [if not exists] 库名;
      ```
   
   2. 修改
      
      ```sql
      rename database 旧库名 to 新库名; // 改名，已废弃
      alter database 库名 character set utf-8;
      ```
   
   3. 删除
      
      ```sql
      drop database [if exists] 库名;
      ```

2. #### 表的管理
   
   1. 创建
      
      ```sql
      create table [if not exists] 表名(
          列名 列类型[(长度) 列的约束],
          列名 列类型[(长度) 列的约束],
          ...
          列名 列类型[(长度) 列的约束]
      );
      ```
      
      ```sql
      // 先不考虑约束
      create table books(
          id int, // 编号
          name varchar(20), // 书名
          authId int, // 作者的编号
          price double, // 价格
          pubilshDate datetime // 出版日期
      );
      ```
   
   2. 修改
      
      ```sql
      // 修改表的列名
      alter table 表名 
      add|drop|modify|change column 列名 参数;
      ```
      
      1. 修改列名（也可以顺带修改列类型）
         
         ```sql
         alter table 表名 change column 列名 新列名 新列类型 [约束];
         ```
      
      2. 修改列的类型或者约束
         
         ```sql
         alter table books modify column 列名 新类型;
         ```
      
      3. 添加新列
         
         ```sql
         alter table author add column 列名 新类型;
         ```
      
      4. 删除列
         
         ```sql
         alter table 表名 drop column 列名;
         ```
      
      5. 修改表名
         
         ```sql
         alter table 旧表名 rename to 新表名;
         ```
   
   3. 删除
      
      ```sql
      drop table [if exists] 表名;
      ```

3. #### 一些常用的建表建库写法
   
   ```sql
   // 在建立数据库时
   drop database if exists 库名;
   create database 库名;
   
   // 在建立表时
   drop table if exists 表名;
   create table 表名() DEFAULT CHARSET=utf8;
   ```

4. #### 表的复制
   
   1. 仅仅复制表的结构
      
      ```sql
      create table 新表名 like 要复制的表名;
      ```
   
   2. 复制表的结构和数据
      
      ```sql
      // 使用子查询
      create table 新表名
      select * from 要复制的表名; // 使用查询可以只复制部分
      ```
   
   3. 仅仅复制表的某些字段
      
      ```sql
      create table 新表名
      select id
      from 要复制的表名
      where 0;
      ```

5. #### 字段的类型
   
   1. 常见的数据类型
      
      1. 数值型
         
         1. 整型
            
            | 类型          | 字节  | 范围                   |
            | ----------- | --- | -------------------- |
            | Tinyint     | 1   | 无符号0~255；有符号-128~127 |
            | Smallint    | 2   | 无符号0~65535           |
            | Mediumint   | 3   |                      |
            | Int、integer | 4   |                      |
            | Bignt       | 8   |                      |
            
            1. 设置无符号和有符号
               
               ```sql
               create table tab_int(
                   t1 int, // 默认有符号
                   t2 int unsigned, // 无符号
                   t3 int(20) zerofill // 当数据长度小于显示长度时以0填充 使用zerofill将会默认无符号
               );
               ```
            
            2. 插入的数据超过范围时
               
               mysql5.6：**超过上限则写入上限，超过下限则写入下限**；
               
               mysql5.6以上：会报错。
            
            3. **长度代表的意义**
               
               整型的长度代表着数据显示的最小长度，小于最小长度的数值会用0补足，但是必须使用zerofill关键字。
            
            4. 如果不设置长度会有默认的长度。
               
               无符号11、有符号10。
         
         2. 小数
            
            | 类型           | 字节  | 范围                                                | 备注  |
            | ------------ | --- | ------------------------------------------------- | --- |
            | float(M,D)   | 4   |                                                   | 浮点数 |
            | double(M,D)  | 8   |                                                   | 浮点数 |
            | decimal(M,D) | M+2 | 最大取值范围与double相同，但是精度更高，给定的decimal的有效取值范围由M和D一起决定。 | 定点数 |
            
            1. M和D的意义
               
               M和D可以省略。decimal默认为(10,0)，float和double会根据插入数值的精度来决定精度。
               
               M：代表整数位加小数位加一起的最大长度，超出则会报错。
               
               D：小数的精度，超出精度会报警告并进行四舍五入，不足会补零。
            
            2. 定点型的精度较高，如果要求插入的数值精度要求入货币运算等，应使用。 
      
      2. 字符型
         
         | 类型               | 最大字符数      | 描述及存储需求                     |
         | ---------------- | ---------- | --------------------------- |
         | char(M)          | M，可以省略默认为1 | M在0~255之间，长度会被尾随空格补足到M，效率较高 |
         | varchar(M)       | M，不可省略     | M在0~65535之间，长度随数据变化，效率较低    |
         | enum(参数1,参数2...) |            | 插入的数据必须在指定参数中               |
         | blob             |            | 长二进制                        |
      
      3. 日期类型
         
         | 类型        | 字节  | 最小值                     | 最大值                 |
         | --------- | --- | ----------------------- | ------------------- |
         | date      | 4   | 1000-01-01              | 9999-12-13          |
         | datetime  | 8   | 1000-01-01 00:00:00     | 9999-12-31 23:59:59 |
         | timestamp | 4   | 0(1970-01-01 00:00:00 ) | 2038年的某一个时刻         |
         | time      | 3   | -838:59:59              | 838:59:59           |
         | year      | 1   | 1901                    | 2155                |
         
         需要注意点：
         
         1. timestamp和datetime的区别
            
            timestamp显示的时间会随着mysql所设置的时区变化而变化，而datetime是一个常量，一旦插入后显示的时间不会随着mysql所设置的时区变化而变化。

6. #### 字段的约束
   
   含义：一种限制，用于限制表中的数据，为了保证表中的数据的准确和可靠性。
   
   ```sql
   create table 表名(
       字段名 字段类型 列级约束，
       字段名 字段类型，
       ...
       表级约束
   )
   ```
   
   1. 约束的类别（六大约束）
      
      1. not null 非空约束
         
         保证该字段的值不能为空
      
      2. default 默认约束
         
         保证该字段有默认值
      
      3. unique 唯一约束
         
         保证该字段的值唯一，**但null可以重复**。
      
      4. primary key 主键约束
         
         用于保证该字段的值非空并且唯一
      
      5. foreign key 外键约束
         
         保证该字段的值必须来自某一个表（包括自己）的某一字段中有的值。
         
         设置在从表中；
         
         关联的字段类型必须一致或者兼容；
         
         主表的关联字段必须为一个key（一般是指主键）。
      
      6. check 检查约束（MySQL中不支持）
         
         保证字段的值必须满足指定的检查条件
      
      7. primary key和unique的对比
         
         |             | 保证唯一性 | 是否允许为空 |
         | ----------- | ----- | ------ |
         | primary key | 1     | 0      |
         | unique      | 1     | 1      |
         |             |       |        |
   
   2. 约束的添加分类
      
      1. 列级约束
         
         所有约束都能写在列级约束中，但外键约束写在列级约束时**无作用**。
      
      2. 表级约束
         
         除了not null和default，其他的约束都能写在表级约束中。
   
   3. 添加约束的时机
      
      1. 创建表时
         
         ```sql
         create table 表名(
             字段名 字段类型 列级约束，
             ...
             [constraint 约束名] 表级约束(要约束的字段名) 约束参数
             # 一般情况下除了外键约束其他的约束都会写为列级约束
         );
         ```
         
         1. 添加列级约束
            
            ```sql
            创建表时直接在字段名和类型后面追加约束类型即可，
            外键约束不支持写在列级约束
            ```
            
            ```sql
            create table stuinfo(
               id int primary key, # 主键约束
               stuName varchar(20) not null, # 非空
               gender char(1) check(gender in ('男', '女')), # msql不支持检查约束
               age int default 18, # 默认约束
            );
            ```
         
         2. 添加表级约束
            
            ```sq;
            在各个字段的最下面
            [constraint 约束名] 约束类型(字段名)
            not null和default只存在于列级约束
            ```
            
            ```sql
            create table stuinfo(
               id int,
               stuname varchar(20),
               gender char(1),
               seat int,
               age int,
               majorid int,
               # 表级约束
               constraint pk primary key(id), # 主键名默认为PRIMARY 即使显示指定也无效
               constraint uq unique(seat), # 唯一键
               constraint fk_stuinfo_major 
               foreign key(marjoid) references major(id) # 外键约束
            );
            ```
      
      2. 修改表时
         
         ```sql
         # 列级
         alter table 表名 modify column 字段名 字段类型 新约束;
         # 表级
         alter table 表名 add [constraint 约束名] 约束类型(字段);
         ```
         
         1. 添加非空约束
            
            ```sql
            alter table 表名 modify column 字段名 字段类型 not null;
            ```
         
         2. 添加默认约束
            
            ```sql
            alter table 表名 modify column 字段名 字段类型 default 默认值;
            ```
         
         3. 添加主键
            
            ```sql
            alter table 表名 modify column 字段名 字段类型 primary key;
            alter table 表名 add primary key(字段名);
            ```
         
         4. 添加唯一
            
            ```sql
            alter table 表名 modify column 字段名 字段类型 unique;
            alter table 表名 add unique(字段名);
            ```
         
         5. 添加外键
            
            ```sql
            alter table 表名 add foreign key(从表字段) refreence 主表(主表字段);
            ```
   
   4. 删除约束
      
      1. 删除非空约束
         
         ```sql
         alter table 表名 modify column 字段名 字段类型 null;
         ```
      
      2. 删除默认约束
         
         ```sql
         alter table 表名 modify column 字段名 字段类型;
         ```
      
      3. 删除主键
         
         ```sql
         alter table 表名 drop primary key;
         ```
      
      4. 删除唯一键
         
         ```sql
         alter table 表名 drop index 约束名;
         ```
      
      5. 删除外键约束
         
         ```sql
         alter table 表名 drop foreign key 外键约束名;
         ```
      
      ```sql
      alter table 表名 modify column 字段名 字段类型; # modify无法删除主键约束
      alter table 表名 drop primary key; # 删除主键
      ```
   
   5. 联合约束
      
      在创建约束时，可以给一个表级约束指定多个字段，这多个字段就形成了联合约束。
      
      ```sql
      [constraint 约束名] 约束类型(字段1,字段2,...)
      ```

7. #### 表示列
   
   概念：又称为自增长列，可以不用手动插入值，系统提供默认序列值。
   
   1. 创建表时设置标识列
      
      ```sql
      create table 表名(
          字段名 字段类型 auto_increment,
          其他字段，
          ...
      );
      ```
   
   2. 修改表示设置表示列
      
      ```sql
      alter table 表名 modify column 字段名 字段类型 auto_increment;f
      ```
   
   3. 控制
      
      使用`show variables like '%auto_incrment%'`可以看到两个变量
      
      | 变量名                      | 默认值 | 说明                 |
      | ------------------------ | --- | ------------------ |
      | auto_increment_increment | 1   | 每次自增长的步长           |
      | auto_increment_offset    | 1   | 自增长的起始值，5.7版本前无法修改 |
   
   4. 特点
      
      1. 标识列只能有一个，并且必须配合key使用，一般配合primary key使用。
      
      2. 标识列的类型只能是数值型
      
      3. 标识列可以通过改变变量设置步长和起始值。
