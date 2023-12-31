# jinja2
jinja2是一种模板语言文件

## 元素
1. 字符串：使用单引号或者双引号
2. 数字：浮点，整型
3. 列表：`[item1,item2,...]`
4. 字典：`{k1:v1,k2:v2}`
5. 布尔：true/false
6. 算术运算：`+ - * / // % **`  `//`为返回整除商
7. 比较运算：`== != > >= < <=`
8. 逻辑运算：`and or not`

## 字面量

## 模板语法
```jinja
{# 注释 #}

{# 取变量属性 #}
{{ var1.attr1 }}

{# 取变量 #}
{{ var1 }}

{# 循环列表 #}
{% for element in list1 %}
{{ element }}
{% endfor %}

{# if条件判断 #}
{# 判断变量是否被定义 #}
{% if var1 is defined %}{# 通过条件判断显示的内容 #}{% endif %}
```


## 过滤器
变量可以通过过滤器进行修改，过滤器与变量之间使用管道符`|`进行分割，可以使用圆括号来传递参数。过滤器可以链式调用，前一个过滤器的输出会作为后一个过滤器的输入
```jinja
{{ list|join(',') }}
```

## 空白控制
默认配置中，模板引擎不会对模板中的空白进行改动。可以通过`-`来删除或块前或者块后的空白
```jinja
{# element将会精密排列并且个元素不换行 #}
{% for element in list1 -%}
    {{ element }}
{%- endfor %}
```

## 转义
```jinja
{{ '{{' }}
{% raw %}
    <ul>
    {% for item in seq %}
        <li>{{ item }}</li>
    {% endfor %}
    </ul>
{% endraw %}
```