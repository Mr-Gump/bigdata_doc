# 第8章 函数

## 8.1 函数简介

**Hive** 会将常用的逻辑封装成函数给用户进行使用，类似于 Java 中的函数。

**好处**：避免用户反复写逻辑，可以直接拿来使用。

**重点**：用户需要知道函数{==叫什么==}，{==能做什么==}。

Hive 提供了大量的内置函数，按照其特点可大致分为如下几类：{++单行函数++}、{++聚合函数++}、{++炸裂函数++}、{++窗口函数++}。

以下命令可用于查询所有内置函数的相关信息。

**1）查看系统内置函数**

```sql
show functions;
```

**2）查看内置函数用法**

```sql
desc function upper;
```

**3）查看内置函数详细信息**

```sql
desc function extended upper;
```

## 8.2 单行函数

单行函数的特点是一进一出，即输入一行，输出一行。

单行函数按照功能可分为如下几类: 日期函数、字符串函数、集合函数、数学函数、流程控制函数等。

### 8.2.1 算术运算函数

| 运算符 | 描述           |
| ------ | -------------- |
| A+B    | A和B 相加      |
| A-B    | A减去B         |
| A*B    | A和B 相乘      |
| A/B    | A除以B         |
| A%B    | A对B取余       |
| A&B    | A和B按位取与   |
| A\|B   | A和B按位取或   |
| A^B    | A和B按位取异或 |
| ~A     | A按位取反      |

案例实操：查询出所有员工的薪水后加 1 显示。

```sql
select sal + 1 from emp;
```

### 8.2.2 数值函数

**1）round：四舍五入**

```sql
select round(3.3);
```

**2）ceil：向上取整**

```sql
select ceil(3.1) ;
```

**3）floor：向下取整**

```sql
select floor(4.8);
```

### 8.2.3 字符串函数

**1）substring：截取字符串**

语法一：substring(string A, int start) 

返回值：string 

说明：返回字符串 A 从 start 位置到结尾的字符串

语法二：substring(string A, int start, int len) 

返回值：string

说明：返回字符串 A 从 start 位置开始，长度为 len 的字符串

案例实操：

（1）获取第二个字符以后的所有字符

```sql
select substring("atguigu",2);
```

输出：

```sql
tguigu
```

（2）获取倒数第三个字符以后的所有字符

```sql
select substring("atguigu",-3);
```

输出：

```sql
igu
```

（3）从第 3 个字符开始，向后获取 2 个字符

```sql
select substring("atguigu",3,2);
```

输出：

```sql
gu
```

**2）replace ：替换**

语法：replace(string A, string B, string C) 

返回值：string

说明：将字符串 A 中的子字符串 B 替换为 C。

```sql
select replace('atguigu', 'a', 'A')
```

输出：

```sql
Atguigu
```

**3）regexp_replace：正则替换**

语法：regexp_replace(string A, string B, string C) 

返回值：string

说明：将字符串 A 中的符合 java 正则表达式 B 的部分替换为 C。注意，在有些情况下要使用转义字符。

案例实操：

```sql
select regexp_replace('100-200', '(\\d+)', 'num') 
```

输出：

```sql
num-num
```

**4）regexp：正则匹配**

语法：字符串 regexp 正则表达式

返回值：boolean

说明：若字符串符合正则表达式，则返回 true，否则返回 false。

（1）正则匹配成功，输出 true

```sql
select 'dfsaaaa' regexp 'dfsa+'
```

输出：

```sql
true
```

（2）正则匹配失败，输出 false

```sql
select 'dfsaaaa' regexp 'dfsb+';
```

输出：

```sql
false
```

**5）repeat：重复字符串**

语法：repeat(string A, int n)

返回值：string

说明：将字符串 A 重复 n 遍。

```sql
select repeat('123', 3);
```

输出：

```sql
123123123
```

**6）split ：字符串切割**

语法：split(string str, string pat) 

返回值：array

说明：按照正则表达式 pat 匹配到的内容分割 str，分割后的字符串，以数组的形式返回。

```sql
select split('a-b-c-d','-');
```

输出：

```sql
["a","b","c","d"]
```

**7）nvl ：替换 null 值**

语法：nvl(A,B) 

说明：若 A 的值不为 null，则返回 A，否则返回 B。 

```sql
select nvl(null,1);
```

输出：

```sql
1
```

**8）concat ：拼接字符串**

语法：concat(string A, string B, string C, ……) 

返回：string

说明：将 A,B,C…… 等字符拼接为一个字符串

```sql
select concat('beijing','-','shanghai','-','shenzhen');
```

输出：

```sql
beijing-shanghai-shenzhen
```

**9）concat_ws：以指定分隔符拼接字符串或者字符串数组**

语法：concat_ws(string A, string…| array(string)) 

返回值：string

说明：使用分隔符 A 拼接多个字符串，或者一个数组的所有元素。

```sql
select concat_ws('-','beijing','shanghai','shenzhen');
```

输出：

```sql
beijing-shanghai-shenzhen
```

**10）get_json_object：解析 json 字符串**

语法：get_json_object(string json_string, string path) 

返回值：string

说明：解析 json 的字符串 json_string，返回 path 指定的内容。如果输入的 json 字符串无效，那么返回 NULL。

案例实操：

**1）获取 json 数组里面的 json 具体数据**

```sql
select get_json_object('[{"name":"大海海","sex":"男","age":"25"},{"name":"小宋宋","sex":"男","age":"47"}]','$.[0].name');
```

输出：

```sql
大海海
```

**（2）获取 json 数组里面的数据**

```sql
select get_json_object('[{"name":"大海海","sex":"男","age":"25"},{"name":"小宋宋","sex":"男","age":"47"}]','$.[0]');
```

输出：

```sql
{"name":"大海海","sex":"男","age":"25"}
```

### 8.2.4 日期函数

1）unix_timestamp：返回当前或指定时间的时间戳

语法：unix_timestamp() 

返回值：bigint 

案例实操：

```sql
select unix_timestamp('2022/08/08 08-08-08','yyyy/MM/dd HH-mm-ss');
```

输出：

```sql
1659946088
```

2）from_unixtime：转化 UNIX 时间戳（从 1970-01-01 00:00:00 UTC 到指定时间的秒数）到当前时区的时间格式

语法：from_unixtime(bigint unixtime[, string format]) 

返回值：string 

案例实操：

```sql
select from_unixtime(1659946088);   
```

输出：

```sql
2022-08-08 08:08:08
```

**3）current_date：当前日期**     

```sql
select current_date;  
```

输出：

```sql
2022-07-11
```

**4）current_timestamp：当前的日期加时间，并且精确的毫秒** 

```sql
select current_timestamp; 
```

输出：

```sql
2022-07-11 15:32:22.402
```

**5）month：获取日期中的月**

语法：month (string date) 

返回值：int 

案例实操：

```sql
select month('2022-08-08 08:08:08');
```

输出：

```sql
8
```

**6）day：获取日期中的日**

语法：day (string date) 

返回值：int 

案例实操：

```sql
select day('2022-08-08 08:08:08')
```

输出：

```sql
8
```

**7）hour：获取日期中的小时**

语法：hour (string date) 

返回值：int 

案例实操：

```sql
select hour('2022-08-08 08:08:08')
```

输出：

```sql
8
```

**8）datediff：两个日期相差的天数（结束日期减去开始日期的天数）**

语法：datediff(string enddate, string startdate) 

返回值：int 

案例实操：

```sql
select datediff('2021-08-08','2022-10-09');   
```

输出：

```sql
-427
```

**9）date_add：日期加天数**

语法：date_add(string startdate, int days) 

返回值：string 

说明：返回开始日期 startdate 增加 days 天后的日期

案例实操：

```sql
select date_add('2022-08-08',2);   
```

输出：

```sql
2022-08-10
```

**10）date_sub：日期减天数**

语法：date_sub (string startdate, int days) 

返回值：string 

说明：返回开始日期 startdate 减少 days 天后的日期。

案例实操：

```sql
select date_sub('2022-08-08',2);
```

输出：

```sql
2022-08-06
```

**11）date_format:将标准日期解析成指定格式字符串**

```sql
select date_format('2022-08-08','yyyy年-MM月-dd日') 
```

输出：

```sql 
2022年-08月-08日
```

