# PromQL

## 表达式语言数据类型
在Promql中表达式或子表达式包括以下四种类型之一：
1. 瞬时向量(Instant vector)：一组时间序列，每个时间序列包含单个样本，它们共享相同的时间戳。即表达式的返回值只会包含该时间序列中的最新的一个样本。这样的表达式称为瞬时向量表达式。
2. 区间向量(Range vector)：一组时间序列，每个时间序列包含一段时间范围内的样本数据。
3. 标量(Scalar)：一个浮点型的数据值
4. 字符串(String)：一个简单的字符串
> 用户输入的表达式返回的数据类型是否合法取决于用例的不同。例如瞬时向量表达式返回的数据类型是唯一可以直接绘制成图标的数据类型。

## 字面量(标量和字符串)
1. 字符串可以用单引号、双引号或者反引号指定为文字常量，反引号中的内容不会进行转义
2. 标量浮点值可以字面上写成`[-](digits)[.(digits)]`

## 向量(指标)
1. `<metrics_name> [{label=value}]`：普通的向量表达式，用于获取某个特定的指标向量
2. `{__name__=<expression>}`：使用内置的__name__，可以使用正则表达式来一次获取多个指标向量

## 时间序列过滤器
1. 瞬时向量过滤器：允许在指定的时间戳内选择一组时间序列和每个时间序列的单个样本值。可以通过向花括号中附加一组标签来进一步过滤`http_requests_total{job="prometheus",group="canary"}`
   1. `=`：选择与提供的字符串完全相同的标签
   2. `!=`：选择与提供的字符串不相同的标签
   3. `=~`：选择正则表达式与提供的字符串(或子字符串)相匹配的的标签
   4. `!~`：选择正则表达式与提供的字符串(或子字符串)不相匹配的的标签
2. 区间向量过滤器：区间向量表达式中需要使用`[]`来定义时间范围
   1. `s`秒
   2. `m`分
   3. `h`时
   4. `d`天
   5. `w`周
   6. `y`年
3. 时间位移操作：`offset`关键字需要紧跟在`{}`选择器后面，来查询过去的一个时间点开始的数据`sum(http_requests_total{method="GET"}[5m] offset 5m)`

## 聚合操作
聚合操作作用于**瞬时向量**。可以将瞬时表达式返回的样本数据进行聚合，形成一个具有较少样本值的新的时间序列
1. 操作符
   1. sum求和
   2. min最小值
   3. max最大值
   4. avg平均值
   5. stddev标准差
   6. stdvar标准差异
   7. count计数
   8. count_values对value个数进行计数
   9. bottomk样本值最小的k个元素
   10. topk样本值最大的k个元素
   11. quantile分布统计
2. 使用方式
   ```
   <aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
   ```
   > 只有count_values、quantile、topk、bottomk支持参数(parameter)。without 用于从计算结果中移除列举的标签，而保留其他标签。by正好相反

## 操作符(运算符)
操作符可以使用于标量之间、向量与标量之间、向量与向量之间的运算，计算逻辑均不同
1. **操作符只能对瞬时向量进行运算**
2. 标量之间的运算得到标量
3. 向量与标量之间的运算，运算符会作用于这个向量的每个样本值上。
4. 向量与向量之间的运算，运算符会一次寻找与左边向量所有标签一致的右边向量进行计算。如果没有匹配元素，则直接丢弃

### 二元运算符
1. 运算符`+` `-` `*` `/` `%` `^`

### 布尔运算符
1. 运算符`==` `!=` `>` `<` `>=` `<=`
2. 在默认情况下，true结果的标签将会保留，运算出false结果的标签将会被舍弃(过滤)
3. 在使用`bool`关键字时`http_requests_total > bool 100`，运算结果将以`0``1`的形式显示
4. 标量之间的运算必须使用`bool`关键字

### 集合运算符
1. 运算符`and` `or` `unless`
2. 集合运算符只能在瞬时向量与瞬时向量之间进行运算
3. `<vector1> and <vector2>`会产生一个新的向量，该向量包含vector1和vector2和可以相互匹配的样本数据
4. `<vector1> or <vector2>`会产生一个新的向量，该向量包含vector1中所有样本数据，以及vector2中无法与vector1相互匹配的样本数据
5. `<vector1> unless <vector2>`会产生一个新的向量，该向量包含vector1中无法与vector2相互匹配的样本数据

### 操作符优先级
1. `^`
2. `*` `/` `%`
3. `+` `-`
4. `==` `!=` `<=` `<` `>=` `>`
5. `and` `unless`
6. `or`

## 匹配模式
向量与向量之间进行操作符运算时会基于左右标签完全匹配的默认规则进行运算，除了默认规则，还可以指定其他规则进行运算。
1. 左右向量的数量一致时：默认匹配模式，运算符会寻找唯一匹配。在两边标签不一致时，可以使用`on(label list)`或者`ignore(label list)`来指定进行匹配的标签或者不进行匹配的标签
   ```
   <vector expr> <bin-op> ignoring(<label list>) <vector expr>
   <vector expr> <bin-op> on(<label list>) <vector expr>
   ```
2. 多对一和一对多：值一侧的向量元素可以与多侧的多个向量元素匹配的情况。在这种情况下，必须使用group修饰符`group_left`或者`group_right`来确定结果向量的样本数与左侧还是右侧向量的样本数一致
   ```
   <vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
   <vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
   <vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
   <vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
   ```
3. 多对一和一对多的情况一定出现在操作符两边的标签不能完全匹配的基础上。所以必须要使用`ignore`和`on`
   > gorup修饰符只能在比较和数学运算中出现，在逻辑运算中默认与右向量中所有元素进行匹配
   > group修饰符可以接受一个标签列表作为参数，表示将对侧的向量的多个标签加入到结果向量中

## 计算函数
| 函数       | 说明                                                                           | 示例                           |
| ---------- | ------------------------------------------------------------------------------ | ------------------------------ |
| rate()     | 专门搭配counter类型数据使用的函数，计算一个时间段，counter值的**每秒**平均增量 | rate()                         |
| increase() | 用来针对Counter这种数据类型，截取其中某一段的增量                              | increase(node_cpu[1m])         |
| sum()      | 对不同标签的key进行求和                                                        | sum(increase(node_cpu[1m]))    |
| topk()     | 只取出时间范围内key的最高k个值，一般用于console查看                            | topk(k,count_wait_connections) |

## 动态标签替换
通过`label_replace`和`label_join`函数，可以自定义的修改向量标签名和对应的标签值
1. `label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)`
   1. v：向量
   2. src_label：源标签名
   3. regex：可以对源标签的值进行正则提取
   4. dst_label：新标签名
   5. replacement：新标签的值，可以使用regex中提取出来的值
2. `label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)`
   1. v：向量
   2. dst_label：新标签名
   3. separator：新标签值的拼接符
   4. src_label_1、src_label_2：源标签名

## 内置函数
1. absent(v instant-vector)：如果传递传递给它的向量参数具有样本数据，则返回空向量；如果传递的向量没有样本数据，则返回不带度量指标名称且带有标签的时间序列，value为1
   ```
   # 这里提供的向量有样本数据
   absent(http_requests_total{method="get"})  => no data
   absent(sum(http_requests_total{method="get"}))  => no data

   # 由于不存在度量指标 nonexistent，所以 返回不带度量指标名称且带有标签的时间序列，且样本值为1
   absent(nonexistent{job="myjob"})  => {job="myjob"}  1
   # 正则匹配的 instance 不作为返回 labels 中的一部分
   absent(nonexistent{job="myjob",instance=~".*"})  => {job="myjob"}  1

   # sum 函数返回的时间序列不带有标签，且没有样本数据
   absent(sum(nonexistent{job="myjob"}))  => {}  1
   ```
2. ceil(v instant-vector)将v中所有元素的样本值四舍五入
3. changes(v instant-vector)返回v内每个样本数据数值的变化次数
   ```
   # 如果样本数据值没有发生变化，则返回结果为 1
   changes(node_load5{instance="192.168.1.75:9100"}[1m]) # 结果为 1
   ```
