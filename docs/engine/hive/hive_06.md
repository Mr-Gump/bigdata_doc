# 第6章 查询

## 6.1 基础语法及执行顺序

1）[:material-link:官网地址](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select)

**2）查询语句语法：**

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference       -- 从什么表查
  [WHERE where_condition]   -- 过滤
  [GROUP BY col_list]        -- 分组查询
   [HAVING col_list]          -- 分组后过滤
  [ORDER BY col_list]        -- 排序    统称为hive中4个by
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
  ]
 [LIMIT number]                -- 限制输出的行数
```

![image-20230213205612309](https://cos.gump.cloud/uPic/image-20230213205612309.png)

## 6.2 基本查询（Select…From）

### 6.2.1 数据准备

（0）原始数据

①在 /opt/module/hive/datas/ 路径上创建 dept.txt 文件，并赋值如下内容：

部门编号 部门名称 部门位置id

```txt title="dept.txt"
10	行政部	1700
20	财务部	1800
30	教学部	1900
40	销售部	1700
```

②在 /opt/module/hive/datas/ 路径上创建 emp.txt 文件，并赋值如下内容：

员工编号 姓名 岗位    薪资  部门

```txt title="emp.txt"
7369	张三	研发	800.00	30
7499	李四	财务	1600.00	20
7521	王五	行政	1250.00	10
7566	赵六	销售	2975.00	40
7654	侯七	研发	1250.00	30
7698	马八	研发	2850.00	30
7782	金九	\N	2450.0	30
7788	银十	行政	3000.00	10
7839	小芳	销售	5000.00	40
7844	小明	销售	1500.00	40
7876	小李	行政	1100.00	10
7900	小元	讲师	950.00	30
7902	小海	行政	3000.00	10
7934	小红明	讲师	1300.00	30
```

（1）创建部门表

```sql
hive (default)>
create table if not exists dept(
    deptno int,    -- 部门编号
    dname string,  -- 部门名称
    loc int        -- 部门位置
)
row format delimited fields terminated by '\t';
```

（2）创建员工表

```sql
create table if not exists emp(
    empno int,      -- 员工编号
    ename string,   -- 员工姓名
    job string,     -- 员工岗位（大数据工程师、前端工程师、java工程师）
    sal double,     -- 员工薪资
    deptno int      -- 部门编号
)
row format delimited fields terminated by '\t';
```

（3）导入数据

```sql
load data local inpath '/opt/module/hive/datas/dept.txt' into table dept;
load data local inpath '/opt/module/hive/datas/emp.txt' into table emp;
```

### 6.2.2 全表和特定列查询

**1）全表查询**

```sql
select * from emp;
```

**2）选择特定列查询**

```sql
select empno, ename from emp;
```

!!! info "注意"

    - SQL 语言{==大小写不敏感==}
    - SQL 可以写在一行或者多行
    - 关键字不能被缩写也不能分行
    - 各子句一般要分行写
    - 使用缩进提高语句的可读性

### 6.2.3 列别名

1）重命名一个列

2）便于计算

3）紧跟列名，也可以在列名和别名之间加入关键字‘AS’ 

4）案例实操

查询名称和部门。

```sql
select 
    ename AS name, 
    deptno dn 
from emp;
```

### 6.2.4 Limit语句

典型的查询会返回多行数据。limit 子句用于限制返回的行数。

```sql
select * from emp limit 5; 
select * from emp limit 2,3; -- 表示从第2行开始，向下抓取3行
```

hive sql 执行过程：

底层不执行 MapReduce 程序，直接抓取数据。

### 6.2.5 Where语句

**1）使用 where 子句，将不满足条件的行过滤掉**

**2）where 子句紧随 from 子句**

**3）案例实操**

查询出薪水大于 1000 的所有员工。

```sql
select * from emp where sal > 1000;
```

!!! warning "注意"

    where 子句中不能使用字段别名。

hive sql 执行过程：

底层不执行 MapReduce 程序，先过滤 sal > 1000 的数据，然后抓取全部数据。

### 6.2.6 关系运算函数

**1）基本语法**

如下操作符主要用于 where、join…on 和 having 语句中。

| 操作符                  | 支持的数据类型 | 描述                                                         |
| ----------------------- | -------------- | ------------------------------------------------------------ |
| A=B                     | 基本数据类型   | 如果A等于B则返回true，反之返回false                          |
| A<=>B                   | 基本数据类型   | 如果A和B都为null，则返回true，如果一边为null，返回false      |
| A<>B, A!=B              | 基本数据类型   | A或者B为null则返回null；如果A不等于B，则返回true，反之返回false |
| A<B                     | 基本数据类型   | A或者B为null，则返回null；如果A小于B，则返回true，反之返回false |
| A<=B                    | 基本数据类型   | A或者B为null，则返回null；如果A小于等于B，则返回true，反之返回false |
| A>B                     | 基本数据类型   | A或者B为null，则返回null；如果A大于B，则返回true，反之返回false |
| A>=B                    | 基本数据类型   | A或者B为null，则返回null；如果A大于等于B，则返回true，反之返回false |
| A [not] between B and C | 基本数据类型   | 如果A，B或者C任一为null，则结果为null。如果A的值大于等于B而且小于或等于C，则结果为true，反之为false。如果使用not关键字则可达到相反的效果。 |
| A is null               | 所有数据类型   | 如果A等于null，则返回true，反之返回false                     |
| A is not null           | 所有数据类型   | 如果A不等于null，则返回true，反之返回false                   |
| in（数值1，数值2）      | 所有数据类型   | 使用 in运算显示列表中的值                                    |
| A [not] like B          | string 类型    | B是一个SQL下的简单正则表达式，也叫通配符模式，如果A与其匹配的话，则返回true；反之返回false。B的表达式说明如下：‘x%’表示A必须以字母‘x’开头，‘%x’表示A必须以字母‘x’结尾，而‘%x%’表示A包含有字母‘x’,可以位于开头，结尾或者字符串中间。如果使用not关键字则可达到相反的效果。 |
| A rlike B, A regexp B   | string 类型    | B是基于java的正则表达式，如果A与其匹配，则返回true；反之返回false。匹配使用的是JDK中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串A相匹配，而不是只需与其字符串匹配。 |

**2）案例实操（between/In/ Is null）**

（1）查询出薪水等于 5000 的所有员工

```sql
select * from emp where sal = 5000;
```

hive sql 执行过程：

底层不执行 MapReduce 程序，先过滤 sal = 5000 的数据，然后抓取全部数据。

（2）查询工资在 500 到 1000 的员工信息

```sql
select * from emp where sal between 500 and 1000;
```

hive sql 执行过程：

底层不执行 MapReduce 程序，先过滤 sal 在 500 到 1000 的数据，然后抓取全部数据。

（3）查询 job 为空的所有员工信息

```sql
select * from emp where job is null;
```

hive sql 执行过程：

底层不执行 MapReduce 程序，先过滤 job 是 null 的数据，然后抓取全部数据。

（4）查询工资是 1500 或 5000 的员工信息

```sql
select * from emp where sal IN (1500, 5000);
```

hive sql 执行过程：

底层不执行 MapReduce 程序，先过滤 sal 是 1500 或者 5000 的数据，然后抓取全部数据。

**3）案例实操（ like 和 rlike ）**

**（1）使用 like 运算选择类似的值**

**（2）选择条件可以包含字符或数字**

- % 代表零个或多个字符（任意个字符）
- _ 代表一个字符

**（3）rlike 子句**

rlike 子句是 Hive 中这个功能的一个扩展，其可以通过 Java 的{==正则表达式==}这个更强大的语言来指定匹配条件。

**（4）案例实操**

①查找名字以"小"开头的员工信息

```sql
select 
    * 
from emp 
where ename like '小%';


select 
    * 
from emp 
where ename rlike '^小';
```

②查找名字以"明"结尾的员工信息

```sql
select 
    * 
from emp 
where ename like '小%';


select 
    * 
from emp 
where ename rlike '^小';
```

③查找名字中带有"明"的员工信息

```sql
select 
    * 
from emp 
where ename like '%明%';


select 
    * 
from emp 
where ename rlike '[明]';
```

### 6.2.7 逻辑运算函数

**1）基本语法（ and / or / not ）**

| 操作符 | 含义   |
| ------ | ------ |
| and    | 逻辑并 |
| or     | 逻辑或 |
| not    | 逻辑否 |

**2）案例实操**

（1）查询薪水大于 1000，部门是 30

```sql
select 
    * 
from emp 
where sal > 1000 and deptno = 30;
```

（2）查询薪水大于 1000，或者部门是 30

```sql
select 
    * 
from emp 
where sal>1000 or deptno=30;
```

（3）查询除了 20 部门和 30 部门以外的员工信息

```sql
select 
    * 
from emp 
where deptno not in(30, 20);
```

### 6.2.8 聚合函数

**1）语法**

- count(*) : 表示统计所有行数，包含 null 值
- count(某列) : 表示该列一共有多少行，不包含 null 值
- max() : 求最大值，不包含 null，除非所有值都是 null
- min() : 求最小值，不包含 null，除非所有值都是 null
- sum() : 求和，不包含 null
- avg() : 求平均值，不包含 null

**2）案例实操**

**（1）求总行数（ count ）**

```sql
select count(*) cnt from emp;
```

hive sql 执行过程：

![image-20230214181448000](https://cos.gump.cloud/uPic/image-20230214181448000.png)

**（2）求工资的最大值（max）**

```sql
select max(sal) max_sal from emp;
```

hive sql 执行过程：

![image-20230214181516342](https://cos.gump.cloud/uPic/image-20230214181516342.png)

**（3）求工资的最小值（min）**

```sql
select min(sal) min_sal from emp;
```

hive sql 执行过程：

![image-20230214181906167](https://cos.gump.cloud/uPic/image-20230214181906167.png)

（4）求工资的总和（sum）

```sql
select sum(sal) sum_sal from emp; 
```

hive sql 执行过程：

![image-20230214181933716](https://cos.gump.cloud/uPic/image-20230214181933716.png)

（5）求工资的平均值（avg）

```sql
select avg(sal) avg_sal from emp;
```

hive sql 执行过程：

![image-20230214181957832](https://cos.gump.cloud/uPic/image-20230214181957832.png)

## 6.3 分组

### 6.3.1 Group By语句

Group By 语句通常会和聚合函数一起使用，按照一个或者多个列队结果进行分组，然后对每个组执行聚合操作。

1）案例实操：

（1）计算 emp 表每个部门的平均工资。

```sql
select 
    t.deptno, 
    avg(t.sal) avg_sal 
from emp t 
group by t.deptno;
```

hive sql 执行过程：

![image-20230214183116136](https://cos.gump.cloud/uPic/image-20230214183116136.png)

（2）计算 emp 每个部门中每个岗位的最高薪水。

```sql
select 
    t.deptno, 
    t.job, 
    max(t.sal) max_sal 
from emp t 
group by t.deptno, t.job;
```

hive sql 执行过程：

![image-20230214183146449](https://cos.gump.cloud/uPic/image-20230214183146449.png)

### 6.3.2 Having语句

**1）having 与 where 不同点**

（1）where 后面不能写分组聚合函数，而 having 后面可以使用分组聚合函数。

（2）having 只用于 group by 分组统计语句。

**2）案例实操**

（1）求每个部门的平均薪水大于 2000 的部门

①求每个部门的平均工资。

```sql
select 
    deptno, 
    avg(sal) 
from emp 
group by deptno;
```

hive sql 执行过程：

![image-20230214183316086](https://cos.gump.cloud/uPic/image-20230214183316086.png)

②求每个部门的平均薪水大于 2000 的部门。

```sql
select 
    deptno, 
    avg(sal) avg_sal 
from emp 
group by deptno  
having avg_sal > 2000;
```

hive sql 执行过程：

![image-20230214183341625](https://cos.gump.cloud/uPic/image-20230214183341625.png)

## 6.4 Join语句

### 6.4.1 等值Join

Hive 支持通常的 sql join 语句，但是{==只支持等值连接，不支持非等值连接==}。

**1）案例实操**

（1）根据员工表和部门表中的部门编号相等，查询员工编号、员工名称和部门名称。

```sql
select 
    e.empno, 
    e.ename, 
    d.dname 
from emp e 
join dept d 
on e.deptno = d.deptno;
```

hive sql 执行过程：

![image-20230214183428787](https://cos.gump.cloud/uPic/image-20230214183428787.png)

### 6.4.2 表的别名

**1）好处**

（1）使用别名可以简化查询。

（2）区分字段的来源。

**2）案例实操**

合并员工表和部门表。

```sql
select 
    e.*,
    d.* 
from emp e 
join dept d 
on e.deptno = d.deptno;
```

### 6.4.3 内连接

内连接：只有进行连接的两个表中都存在与连接条件相匹配的数据才会被保留下来。

```sql
select 
    e.empno, 
    e.ename, 
    d.deptno 
from emp e 
join dept d 
on e.deptno = d.deptno;
```

### 6.4.4 左外连接

左外连接：join 操作符左边表中符合 where 子句的所有记录将会被返回。

```sql
select 
    e.empno, 
    e.ename, 
    d.deptno 
from emp e 
left join dept d 
on e.deptno = d.deptno;
```

### 6.4.5 右外连接

右外连接：join 操作符右边表中符合 where 子句的所有记录将会被返回。

```sql
select 
    e.empno, 
    e.ename, 
    d.deptno 
from emp e 
right join dept d 
on e.deptno = d.deptno;
```

### 6.4.6 满外连接

满外连接：将会返回所有表中符合 where 语句条件的所有记录。如果任一表的指定字段没有符合条件的值的话，那么就使用 null 值替代。

```sql
select 
    e.empno, 
    e.ename, 
    d.deptno 
from emp e 
full join dept d 
on e.deptno = d.deptno;
```

### 6.4.7 多表连接

!!! info "注意"

    连接 n 个表，至少需要 n - 1 个连接条件。例如：连接三个表，至少需要两个连接条件。

数据准备，在 /opt/module/hive/datas/ 下创建 location.txt

部门位置id  部门位置

```txt title="location.txt"
1700	北京
1800	上海
1900	深圳
```

**1）创建位置表**

```sql
create table if not exists location(
    loc int,           -- 部门位置id
    loc_name string   -- 部门位置
)
row format delimited fields terminated by '\t';
```

**2）导入数据**

```sql
load data local inpath '/opt/module/hive/datas/location.txt' into table location;
```

**3）多表连接查询**

```sql
select 
    e.ename, 
    d.dname, 
    l.loc_name
from emp e 
join dept d
on d.deptno = e.deptno 
join location l
on d.loc = l.loc;
```

大多数情况下，Hive 会对每对 join 连接对象启动一个 MapReduce 任务。本例中会首先启动一个 MapReduce job 对表 e 和表 d 进行连接操作，然后会再启动一个 MapReduce job 将第一个 MapReduce job 的输出和表 l 进行连接操作。

!!! info "注意"

    为什么不是表 d 和表 l 先进行连接操作呢？这是因为 Hive 总是按照从左到右的顺序执行的。

### 6.4.8 笛卡尔集

**1）笛卡尔集会在下面条件下产生**

（1）省略连接条件

（2）连接条件无效

（3）所有表中的所有行互相连接

**2）案例实操**

```sql
select 
    empno, 
    dname 
from emp, dept;
```

hive sql 执行过程：

![image-20230214184215824](https://cos.gump.cloud/uPic/image-20230214184215824.png)

### 6.4.9 联合（union & union all）

**1）union & union all 上下拼接**

union 和 union all 都是上下拼接 sql 的结果，这点是和 join 有区别的，join 是左右关联，union 和 union all 是上下拼接。{==union 去重，union all 不去重==}。

union 和 union all 在上下拼接 sql 结果时有两个要求：
（1）两个 sql 的结果，列的个数必须相同

（2）两个 sql 的结果，上下所对应列的类型必须一致

**2）案例实操**

将员工表 30 部门的员工信息和 40 部门的员工信息，利用 union 进行拼接显示。

```sql
select 
    *
from emp
where deptno=30
union
select 
    *
from emp
where deptno=40;
```

## 6.5 排序

### 6.5.1 全局排序（Order By）

Order By：全局排序，只有一个 Reduce。

**1）使用 Order By 子句排序**

asc（ascend）：升序（默认）

desc（descend）：降序

**2）Order By 子句在 select 语句的结尾**

**3）基础案例实操** 

（1）查询员工信息按工资升序排列

```sql
select 
    * 
from emp 
order by sal;
```

hive sql 执行过程：

![image-20230214184534482](https://cos.gump.cloud/uPic/image-20230214184534482.png)

（2）查询员工信息按工资降序排列

```sql
select 
    * 
from emp 
order by sal desc;
```

**4）按照别名排序案例实操**

按照员工薪水的 2 倍排序。

```sql
select 
    ename, 
    sal * 2 twosal 
from emp 
order by twosal;
```

hive sql 执行过程：

![image-20230214184614533](https://cos.gump.cloud/uPic/image-20230214184614533.png)

**5）多个列排序案例实操**

按照部门和工资升序排序。

```sql
select 
    ename, 
    deptno, 
    sal 
from emp 
order by deptno, sal;
```

hive sql 执行过程：

![image-20230214184637570](https://cos.gump.cloud/uPic/image-20230214184637570.png)

### 6.5.2 每个Reduce内部排序（Sort By）

Sort By：对于大规模的数据集 order by 的效率非常低。在很多情况下，并不需要全局排序，此时可以使用 Sort by。

Sort by 为每个 reduce 产生一个排序文件。每个 Reduce 内部进行排序，对全局结果集来说不是排序。

**1）设置 reduce 个数**

```sql
set mapreduce.job.reduces=3;
```

**2）查看设置 reduce 个数**

```sql
set mapreduce.job.reduces;
```

**3）根据部门编号降序查看员工信息**

```sql
select 
    * 
from emp 
sort by deptno desc;
```

hive sql 执行过程：

![image-20230214184850096](https://cos.gump.cloud/uPic/image-20230214184850096.png)

**4）将查询结果导入到文件中（按照部门编号降序排序）**

```sql
insert overwrite local directory '/opt/module/hive/datas/sortby-result'
 select * from emp sort by deptno desc;
```

### 6.5.3 分区（Distribute By）

Distribute By：在有些情况下，我们需要控制某个特定行应该到哪个 Reducer，通常是为了进行后续的聚集操作。distribute by 子句可以做这件事。distribute by 类似 MapReduce 中 partition（自定义分区），进行分区，结合 sort by 使用。 

对于 distribute by 进行测试，一定要分配多 reduce 进行处理，否则无法看到 distribute by 的效果。

**1）案例实操：**

（1）先按照部门编号分区，再按照员工编号薪资排序

```sql
set mapreduce.job.reduces=3;

insert overwrite local directory '/opt/module/hive/datas/distribute-result' 
select 
    * 
from emp 
distribute by deptno 
sort by sal desc;
```

!!! info "注意"

    - distribute by 的分区规则是根据分区字段的 hash 码与 reduce 的个数进行相除后，余数相同的分到一个区
    
    - Hive 要求 distribute by 语句要写在 sort by 语句之前
    - 演示完以后 mapreduce.job.reduces 的值要设置回 -1，否则下面分区 or 分桶表 load 跑 MapReduce 的时候会报错

hive sql 执行过程：

![image-20230214185948620](https://cos.gump.cloud/uPic/image-20230214185948620.png)

### 6.5.4 分区排序（Cluster By）

当 distribute by 和 sort by 字段相同时，可以使用 cluster by 方式。

cluster by 除了具有 distribute by 的功能外还兼具 sort by 的功能。但是排序{==只能是升序排序==}，不能指定排序规则为 asc 或者 desc。

（1）以下两种写法等价

```sql
select 
    * 
from emp 
cluster by deptno;
 
select 
    * 
from emp 
distribute by deptno 
sort by deptno;
```

!!! info "注意"

    按照部门编号分区，不一定就是固定死的数值，可以是 20 号和 30 号部门分到一个分区里面去。

hive sql 执行过程：

![image-20230214190129518](/Users/mrgump/Library/Application Support/typora-user-images/image-20230214190129518.png)
