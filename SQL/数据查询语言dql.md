# 数据库查询语言DQL

1. #### 基础查询
   
   ```sql
   select 查询列表 from 表名;
   ```
   
   特点：
   
   1. 查询列表可以是：表中的字段、常量值、表达式、函数。
   
   2. 查询的结果是一个虚拟的表格。

2. #### 条件查询
   
   ```sql
   select 查询列表 from 表名 where 查询条件;
   ```
   
   筛选条件的分类：
   
   1. 按条件表达式进行筛选
      
      `>` `<` `=` `!=` `<>` `>=` `<=` `<=>`
   
   2. 按逻辑表示筛选
      
      `and` `or` `not`
   
   3. 复杂条件表达式
      
      `like` `between and` `in` `is null` `is not null`
   
   4. 通配符（搭配like进行使用）
      
      `%` 任意多个字符，包含0个字符
      
      `-` 任意单个字符
      
      使用`\`转义
   
   5. 注意事项
      
      between xxx and xxx 包含临界词
      
      in 列表的值类型必须统一或者兼容
      
      <=> 安全等于，既可以判断普通类型，也可以判断null
      
      not 对关键字仅支持 `between and` `in` `exists`

3. #### 排序查询
   
   ```sql
   select 查询列表 from 表名 order by 排序列表 desc|asc
   ```
   
   1. 排序列表可以是：表中的字段、表达式、函数。
   
   2. `asc`升序，`desc`降序，不写默认升序
   
   3. 给表达式添加别名后可以按别名进行排序
   
   4. 排序支持按多个字段进行排序，`,`隔开，可以指定降序升序
   
   5. order by一般是在查询的最后面

4. #### 常见函数
   
   ```sql
   select 函数名(实参列表) [from 表名];
   ```
   
   1. 分类
      
      1. 单行函数
         
         如concat、length、isnull等
         
         1. 字符函数
            
            1. length()      返回参数的**字节**大小
            
            2. concat()      拼接字符串，如果传入的值有null直接返回null
            
            3. upper()  lower()  转为大、小写
            
            4. substr(str, index)  截取从index开始（包括）后的所有字符串，**sql的索引是从1开始**
               
               substr(str, start, length)  截取从start开始（包括）后的length长（字符长度）的字符串
            
            5. instr(str, substr)   返回substr第一次在str中出现的起始索引，没有返回0
            
            6. trim(str)   去掉str中的前后空格
               
               trim(substr from str)  去掉str中的前后substr
            
            7. lpad(str, length, substr)     用substr左填充str至length长度，str过长会从左边截断
               
               rpad(str, length, substr)
            
            8. replace(str, substr1, substr2)   将str中的substr1全换成substr2
         
         2. 数学函数
            
            1. round(double)   四舍五入至整数
               
               round(double, int)   四舍五入至int位小数点
            
            2. ceil(double)   向上取整
               
               floor()   向下取整
               
               truncate(double, int)   截断至int位小数
            
            3. mod()   取余
         
         3. 日期函数
            
            1. now()   返回系统当前日期和时间
            
            2. curdate()   只返回系统日期
            
            3. curtime()   只返回系统时间
            
            4. year(date)  返回传入date的年
               
               month(date)  月
               
               monthname(date)  英文月
               
               day 日
            
            5. str_to_date(str, format)   将str用format转为日期类型
            
            6. date_format(date, format)   将日期类型转为指定format的str
            
            7. datediff(date1, date2)   date1-date2返回天数
         
         4. 其他函数
            
            version()  查看版本号
            
            datebase()  查看当前版本
            
            user()  查看当前用户
         
         5. 流程控制函数
            
            1. if(逻辑表达式, 表达式2, 表达式3) 逻辑表达式成立返回表达式2值否则返回表达式3值
            
            2. case  when
               
               简单case函数
               
               ```sql
               # 用法1
               case 要判断的表达式 
               when 常量1 then 要显示值或语句 # case后的值等于常量时触发
               when 常量2 then 要显示值或语句
               when 常量3 then 要显示值或语句
               ...
               []
               end
               ```
               
               ```sql
               # 例子
               select 
               case department_id 
               when 90 then '部门名1'
               when 100 then '部门名2'
               else '部门名3'
               end as department_name
               from employees
               where last_name = 'k_ing';
               ```
               
               case搜索函数
               
               ```sql
               # 用法2
               case 
               when 逻辑表达式1 then 要显示的值或语句
               when 逻辑表达式2 then 要显示的值或语句
               when 逻辑表达式3 then 要显示的值或语句
               ...
               [else 要显示的值或语句]
               end
               ```
               
               ```sql
               # 例子
               select 
               case 
               when department_id=90 then '部门1'
               when department_id=100 then '部门2'
               else '部门3'
               end as department_name
               from employees
               where last_name='k_ing';
               ```
      
      2. 分组函数（做统计功能使用，又称为统计函数、聚合函数、组函数）
         
         1. 分类
            
            sum求和、avg平均值、max最大值、min最小值、count统计数据个数
         
         2. 简单的使用
            
            ```sql
            select sum(salary) from emplpyees;
            select avg(salary) from emplpyees;
            select max(salary) from emplpyees;
            select min(salary) from emplpyees;
            select count(salary) from emplpyees;
            ```
         
         3. 支持的参数类型
            
            sum、avg一般用于处理数值型。max、min、count可以处理任何类型
         
         4. 是否忽略null
            
            都忽略null值
         
         5. 可以和distinct搭配实现去重运算
         
         6. 一般使用count(*)来统计行数 
         
         7. 和分组查询一同查询的字段要求是group by后的字段

5. #### 分组函数
   
   ```sql
   select column, group_function(cloumn) group by group_by_expression;
   # 注意：5.7之后分组查询只能select聚合函数和group by后面涉及的字段
   ```
   
   1. 简单的查询
      
      ```sql
      select avg(salary), job_id from employees;
      ```
   
   2. 添加复杂的筛选条件
      
      ```sql
      // 查询哪个部门的员工数量大于2(分组后进行筛选)(使用having关键字)
      select department_id,count(*) as dnum
      from employees
      group by department_id
      having dnum > 2;
      ```
      
      ```sql
      // 查询有奖金员工最大工资大于12000的工种的的编号和最高工资
      select max(salary) as maxs,job_id
      from employees
      where commission_pct is not null
      group by job_id
      having maxs > 12000;
      ```
      
      ```sql
      select count(*) as c, length(last_name) as len_name
      from employees
      group by len_name
      having c > 5
      order by c;
      ```
   
   3. 分组的筛选分为两类，一类是分组前筛选，一类是分组后筛选
      
      |       | 数据源     | 位置          | 关键字    |
      | ----- | ------- | ----------- | ------ |
      | 分组前筛选 | 原始表     | group by 前面 | where  |
      | 分组后筛选 | 分组后的结果集 | group by 后面 | having |
      
      能使用分组前筛选的尽量使用分组前筛选
      
      group by 支持多个字段排序
      
      排序放在整个分组之后

6. #### 连接查询（多表查询）
   
   含义：当需要查询的字段来自多个表时，就需要用到多表查询
   
   当使用下列查询语句时会发生笛卡尔乘积现象，是因为没有指定连接条件。
   
   ```sql
   select last_name,job_tible from employees,jobs;
   ```
   
   所以需要指定过滤条件
   
   ```sql
   select last_name,job_title from employees,jobs 
   where employees.job_id=jobs_job_id;
   ```
   
   连接查询有很多种类：
   
   按年代来分：
   
   1. sql92标准：支持内连接
      
      ```sql
      // 语法
      select 查询列表
      from 表1 别名, 表2 别名
      where 筛选条件（包括连接筛选条件）
      group by 分组条件
      having 分组后筛选条件
      order by 排序条件
      ```
   
   2. sql99标准【推荐】：支持内连接+外连接（左外右外）+交叉连接
      
      ```sql
      // 语法
      select 查询列表
      from 表1 别名 <join type> join 表2 别名 //join type类型必填
      on 连接条件
      where 筛选条件
      group by 分组条件
      having 分组后筛选条件
      order by 排序条件
      ```
   
   按功能分类：
   
   1. 内连接（92语法）
      
      1. 自连接
         
         ```sql
         // 查询员工和他上级的名字
         select e1.last_name, e2.last_name
         ```
      
      2. 等值连接（显示两个表的交集部分）
         
         ```sql
         // 查询员工名和对应的部门名
         select last_name,department_name
         from employees,departments
         where employees.department_id = departments.department_id;
         ```
         
         ```sql
         // 当表名比较长时可以为表起别名, 起了别名之后不能再使用原来的名字
         select e.last_name, e.job_id, j.job_title
         from employees as e, jobs as j
         where e.job_id= j.job_id;
         ```
         
         多表等值连接的结果为多表连接的交集部分
         
         n表连接，至少需要n-1个连接条件
         
         多表顺序没有要求
         
         一般需要为表起别名
      
      3. 非等值连接
         
         ```sql
         // 查询出员工的工资和工资界别
         select salary, grade_level
         from employees as e, job_grades as j
         where e.salary between j.lowest_sal and j.highest_sal; 
         ```
   
   2. 内连接（99语法）innner
      
      ```sql
      select 查询列表
      from 表1 别名 inner join 表2 别名 // inner可以省略
      on 连接条件;
      ```
      
      1. 自连接
         
         ```sql
         select e1.last_name, e2.last_name
         ```
      
      2. 等值连接
         
         ```sql
         // 查询每个员工的部门名
         select last_name, department_name
         from employees e 
         inner join departments d
         on e.department_id = d.department_id;
         ```
         
         ```sql
         // 查询部门个数大于3的城市名和具体部门个数
         select city, count(*) as c
         from locations l inner join departments d
         on l.location_id = d.location_id
         group by l.city
         having c > 3;
         ```
         
         ```sql
         // 查询员工个数大于3的部门名和具体员工数，按个数降序排序
         select department_name, count(*) c
         from departments d inner join employees e
         on d.department_id = e.department_id 
         group by d.department_id
         having  c > 3
         order by c desc;
         ```
         
         ```sql
         // 查询员工名、部门名、工种名，并按部门名排序
         select last_name, department_name as dn, job_title
         from employees e 
         inner join departments d on e.department_id = d.department_id
         inner join jobs j on e.job_id = j.job_id
         order by dn desc;
         ```
      
      3. 非等值连接
         
         ```sql
         select e.last_name, salary, grade_level `level`
         from employees e join job_grades g
         on e.salary between g.lowest_sal and g.highest_sal
         order by `level`;
         ```
   
   3. 外连接（99语法）
      
      适用于一个表中有但是连接的表中没有的记录 
      
      > 特点：
      > 
      > 1. 外连接的查询结果为主表中所有的记录，不显示从表中不符合匹配规则的记录
      > 
      > 2. 左外右外交换顺序能实现一样的结果
      
      1. 左外连接 left [outer]
         
         left左边的是主表
         
         ```sql
         // 查询男朋友不在男神表中的女神名
         select name, Boyname
         from beauty b left join boys
         on b.boyfriend_id = boys.id
         where boys.id is null;
         ```
         
         ```sql
         // 查询哪个部门没有员工
         select d.department_id, department_name, last_name
         from departments d left join employees e
         on d.department_id = e.department_id
         where e.employee_id is null;
         ```
      
      2. 右外连接 right [outer]
         
         right右边的是主表
         
         ```sql
         // 查询男朋友不在男神表中的女神名
         select name, Boyname
         from boys right join beauty b
         on b.boyfriend_id = boys.id
         where boys.id is null;
         ```
      
      3. 全外连接 full [outer]
         
         sql5.7是不支持全外连接
         
         全外连接查询出的结果是两张表的并集 
   
   4. 交叉连接（99语法） cross
      
      ```sql
      select b.*, bo.*
      from beauty b
      cross join boys bo;
      ```

7. #### 七种join查询
   
   ```sql
   # 1.inner join  AB共有部分
   select * from A inner join B 
   on A.a = B.b;
   # 2.left join   A全部B共有
   select * from A left join B
   on A.a = B.b;
   # 3.right join  A共有B全部
   select * from A right join B
   on A.a = B.b;
   # 4.A有B无
   select * from A left join B
   on A.a = B.b
   where B.b is null;
   # 5.A无B有
   select * from A right join B
   on A.a = B.b
   where A.a is null;
   # 6.笛卡尔积
   select * from A outer join B
   on A.a = B.b;
   # 7.A有B无或A无B有
   select * from A outer join B
   on A.a = B.b
   where A.a is null or B.b is null;
   ```

8. #### 子查询
   
   概念：出现在一个语句内的select语句被称为子（内）查询。对应地，外部的查询语句成为主（内）查询。
   
   **分类：**
   
   1. 按子查询出现的位置：
      
      select后面：仅仅支持标量子查询
      
      from后面：支持表子查询
      
      where或者having后面：支持标量子查询或者列子查询（也支持行子查询单用的较少）
      
      exists后面（相关子查询）：支持表子查询
   
   2. 按结果集的行列数不同：
      
      1. 标量子查询（结果集只有一行一列）
      
      2. 列子查询（结果集多行一列）
      
      3. 行子查询（结果集一行多列）
      
      4. 表子查询（结果集一般为多行多列）
      
      5. Exists子查询（返回值为0或1）

9. 使用：
   
   1. where后面的标量子查询
      
      > 标量子查询一般搭配单行操作符 `>` `<` `=`等使用
      
      ```sql
      // 查询工资大于Able的员工姓名
      select last_name, salary
      from employees
      where salary > (
         select salary where last_name = 'Abel'
      );
      ```
   
   2. where后面的列子查询
      
      > 一般搭配多行操作符 `in` `not in` `any` `some` `all` 在where关键字后使用
      
      ```sql
      // 查询所在部门的地址为1400或1700的员工
      select last_name
      from employees
      where department_id in (
          select distinct department_id
          from departments
          where location_id in (1400, 1700)
      );
      ```
   
   3. where后面的行子查询
      
      ```sql
      // 查询员工编号最小同时员工工资最高的员工
      select last_name
      from employees
      where (employee_id, salary) = (
          select min(employee_id), max(salary)
          from employees
      );
      ```
   
   4. 放在select后的子查询
      
      ```sql
      // 查询每个部门的员工个数
      select d.*, (
          select count(*)
          from employees e
          where e.department_id = d.department_id
      ) 个数
      from departments d;
      ```
   
   5. 放在from后面的子查询
      
      ```sql
      // 查询每个部门的平均工资的工资等级
      select avg.di, avg.av, sg.grade_level
      from (
          select avg(salary) av, department_id di
          from employees
          group by department_id
      ) avg join job_grades sg
      on av between sglowest_sal and sg.highest_sal;
      ```
      
      > 将子查询作为查询表必须起别名
   
   6. 放在exists后面（相关子查询）
      
      ```sql
      // 查询没有女朋友的男生信息
      select bo.*
      from boys bo 
      where not exists (
          select *
          from beauty b
          where bo.id = b.boyfriend_id
      );
      ```

10. #### 分页查询
    
    ```sql
    select 查询列表
    from 表 [join]
    where
    group by
    having
    order by 
    limit offset,size;
    ```
    
    > offset：要显示条目的起始索引（**0开始**）
    > 
    > size：条目个数
    > 
    > 一般配合排序使用，放在查询语句的最后
    
    ```sql
    // 查询11条到25条
    select * from employees
    limit 10,15;
    ```
    
    ```sql
    // 显示第page页，每页的条目数size
    select * from employees
    limit (page-1)*size, size;
    ```

11. #### 联合查询 union
    
    概念：将多条查询语句的结果合并为一个结果
    
    语法：
    
    ```sql
    查询语句1 
    union [all]
    查询语句2
    union [all]
    ...
    ```
    
    > 特点：
    > 
    > 1. 所有的查询的列数必须相同，最后结果以第一个结果的字段名为准。
    > 
    > 2. 适用于要查询的结果来自于多个表，且这多个表没有直接的连接关系，但查询的信息一致。
    > 
    > 3. 默认自动去重，使用关键字all实现不去重。
    
    ```sql
    // 查询部门编号大于90或邮箱包含a的员工信息
    select * from employees where email like '%a'
    ```
