# 事务控制语言TCL

Transcation Control Language

1. #### 事务
   
   1. 概念：一个或一组sql语句组成执行单元，称为事务，这个执行单元要么全部执行要么全部失败。innodb引擎支持事务。
   
   2. 特性（ACID）
      
      1. 原子性（Atomicity）
         
         原子性是指事务是一个不可分割的工作单位，事务中的操作要么全发生，要么都不发生。
      
      2. 一致性（Consistency）
         
         事务必须使数据库从一个一致性状态变换到另一个一致性状态。
      
      3. 隔离性（Isolation）
         
         事务的执行不能被另一个事务干扰，即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发的事务不能相互干扰。（根据隔离级别决定）
      
      4. 持久性（Durability）
         
         持久性是指一个事务一旦被提交，它对数据库中的数据改变就是永久性的，接下来的其他操作和数据库故障不应该对其有任何影响。 

2. #### 事务的创建
   
   1. 隐式事务：事务没有明显的开启和结束的标记
      
      比如insert、update、delete语句。
   
   2. 显示事务：事务有明显的开启和结束标记
      
      前提：禁用自动提交功能 `set autocommit=0;` 这个操作只在当前会话有效。
   
   3. 步骤
      
      1. 开启事务
         
         ```sql
         set autocommit=0;
         start transaction; # 可选
         ```
      
      2. 编写sql语句
         
         一般是指select、insert、update、delete，DDL语言不适用于事务
      
      3. 结束事务
         
         ```sql
         # 需要配合数据库驱动编程来实现根据情况判断提交事务还是回滚事务
         commit; # 提交事务
         rollback; # 回滚事务
         ```
   
   4. 例子
      
      ```sql
      set autocommit=0;
      start transaction;
      update 表名 set xxx where xxx ;
      commit;
      ```

3. #### 事务的并发
   
   当同时运行多个事务，当这些事务同时访问数据库中相同的数据时，如果没有采取必要的隔离机制，就会导致各种并发问题。
   
   1. 可能发生的问题
      
      1. 脏读：事务A修改了数据但是还未提交，事务B读取这个数据，事务A回滚，事务B读到的数据其实只是个临时数据。
      
      2. 不可重复读：事务A读取了某个数据还未提交，事务B修改了这个数据并提交，事务A再次读此数据，产生了两次不一样的结果。
      
      3. 幻读：事务A增删一个表中某几行的数据还未提交，事务B插入了新的数据，事务A提交时修改的数据超出预期，产生幻读。
   
   2. 查看默认隔离级别
      
      ```sql
      select @@transaction_isolation;
      ```
   
   3. 隔离级别的分类
      
      | 隔离级别                       | 描述                                                                                                         |
      |-----------------------------|------------------------------------------------------------------------------------------------------------|
      | read uncommitted（读未提交数据） | 允许事务读取未被其他事务提交的变更。脏读、不可重复读和幻读问题都会出现                                         |
      | read committed（读已提交数据）   | 只允许事务读取已经被其他事务提交的变更。可避免脏读，不可重复读和幻读问题仍会出现。                              |
      | repeatable-read（可重复读）      | 确保事务可以多次从一个字段中读取到相同的值，在这个事务持续期间，禁止其他事务对这个字段进行更新。仍无法避免幻读。 |
      | serializable（串行化）           | 确保事务在对某个表进行操作时其他事务无法对这个表进行增删改操作，所有并发问题均可解决，但效率极其低下。          |
   
   4. 设置隔离级别
      
      ```sql
      set session transaction isolation level 
      read uncommitted | read committed | repeatable read | serializable;
      #mysql中默认 repeatable read
      ```
   
   5. 不同的隔离级别能解决的问题
      
      | 能否预防         | 脏读 | 不可重复读 | 幻读 |
      |------------------|------|------------|------|
      | read uncommitted | ×    | ×          | ×    |
      | read committed   | √    | ×          | ×    |
      | repeatable read  | √    | √          | ×    |
      | serializable     | √    | √          | √    |

4. #### 事务节点
   
   1. 节点概念：在事务中，使用`savepoint 节点名`的语句可以设置一个事务节点。
   
   2. 语法
      
      ```sql
      set autocommit=0;
      start transaction;
      delete from xxx where xxx=x;
      savepoint a; # 设置保存点
      delete from xxx where xxx=x;
      rollback to a; # 可以只回滚到设置a保存点时的数据库状态
      commit;
      ```
