# 第10章 分区表和分桶表

## 10.1 分区表

Hive 中的分区就是把一张大表的数据按照业务需要分散的存储到{==多个目录==}，每个目录就称为该表的一个分区。在查询时通过 where 子句中的表达式选择查询所需要的分区，这样的查询效率会提高很多。

### 10.1.1 分区表基本语法

1. 创建分区表

```sql
create table dept_partition
(
    deptno int,    --部门编号
    dname  string, --部门名称
    loc    string  --部门位置
)
    partitioned by (day string)
    row format delimited fields terminated by '\t';
```

2. 分区表读写数据

1）写数据

**（1）load**

1⃣️数据准备

在 /opt/module/hive/datas/ 路径上创建文件 dept_20220401.log，并输入如下内容。

```log title="dept_20220401.log"
10	行政部	1700
20	财务部	1800
```

2⃣️装载语句

```sql
load data local inpath '/opt/module/hive/datas/dept_20220401.log' 
into table dept_partition 
partition(day='20220401');
```

**（2）insert**

将 `day='20220401'` 分区的数据插入到 `day='20220402'` 分区，可执行如下装载语句

```sql
insert overwrite table dept_partition partition (day = '20220402')
select deptno, dname, loc
from dept_partition
where day = '2020-04-01';
```

**2）读数据**

查询分区表数据时，可以将分区字段看作表的{==伪列==}，可像使用其他字段一样使用分区字段。

```sql
select deptno, dname, loc ,day
from dept_partition
where day = '2020-04-01';
```

**3. 分区表基本操作**

1）查看所有分区信息

```sql
show partitions dept_partition;
```

2）增加分区

（1）创建单个分区

```sql
alter table dept_partition 
add partition(day='20220403');
```

（2）同时创建多个分区（分区之间不能有逗号）

```sql
alter table dept_partition 
add partition(day='20220404') partition(day='20220405');
```

3）删除分区

（1）删除单个分区

```sql
alter table dept_partition 
drop partition (day='20220403');
```

（2）同时删除多个分区（分区之间必须有逗号）

```sql
alter table dept_partition 
drop partition (day='20220404'), partition(day='20220405');
```

4）修复分区

Hive 将分区表的所有分区信息都保存在了元数据中，只有元数据与 HDFS 上的分区路径一致时，分区表才能正常读写数据。若用户手动创建/删除分区路径，Hive 都是感知不到的，这样就会导致 Hive 的元数据和 HDFS 的分区路径不一致。再比如，若分区表为外部表，用户执行 drop partition 命令后，分区元数据会被删除，而 HDFS 的分区路径不会被删除，同样会导致 Hive 的元数据和 HDFS 的分区路径不一致。

若出现元数据和 HDFS 路径不一致的情况，可通过如下几种手段进行修复。

（1）add partition

若手动创建 HDFS 的分区路径，Hive 无法识别，可通过 add partition 命令增加分区元数据信息，从而使元数据和分区路径保持一致。

（2）drop partition

若手动删除 HDFS 的分区路径，Hive 无法识别，可通过 drop partition 命令删除分区元数据信息，从而使元数据和分区路径保持一致。

（3）msck

若分区元数据和 HDFS 的分区路径不一致，还可使用 msck 命令进行修复，一下是改命令的用法说明。

```sql
msck repair table table_name [add/drop/sync partitions];
```

!!! info "说明"
    msck repair table table_name add partitions：该命令会增加 HDFS 路径存在但元数据缺失的分区信息。
    msck repair table table_name drop partitions：该命令会删除 HDFS 路径已经删除但元数据仍然存在的分区信息。
    msck repair table table_name sync partitions：该命令会同步 HDFS 路径和元数据分区信息，相当于同时执行上述的两个命令。
    msck repair table table_name：等价于 msck repair table table_name add partitions 命令。

###     10.1.2 二级分区表

!!! question "思考"

    如果一天内的日志数据量也很大，如何再将数据拆分?答案是二级分区表，例如可以在按天分区的基础上，再对每天的数据按小时进行分区。

1）二级分区表建表语句

```sql
create table dept_partition2(
    deptno int,    -- 部门编号
    dname string, -- 部门名称
    loc string     -- 部门位置
)
partitioned by (day string, hour string)
row format delimited fields terminated by '\t';
```

2）数据装载语句

```sql
load data local inpath '/opt/module/hive/datas/dept_20220401.log' 
into table dept_partition2 
partition(day='20220401', hour='12');
```

3）查询分区数据

```sql
select 
    * 
from dept_partition2 
where day='20220401' and hour='12';
```

### 10.1.3 动态分区

动态分区是指向分区表 insert 数据时，被写往的分区不由用户指定，而是由每行数据的最后一个字段的值来动态的决定。使用动态分区，可只用一个 insert 语句将数据写入多个分区。

1）动态分区相关参数

（1）动态分区功能总开关（默认 {==true==}，开启）

```sql
set hive.exec.dynamic.partition=true
```

（2）严格模式和非严格模式

动态分区的模式，默认 strict（严格模式），要求必须指定至少一个分区为静态分区，nonstrict（非严格模式）允许所有的分区字段都使用动态分区。

```sql
set hive.exec.dynamic.partition.mode=nonstrict
```

（3）一条 insert 语句可同时创建的最大的分区个数，默认为 1000。

```sql
set hive.exec.max.dynamic.partitions=1000
```

（4）单个 Mapper 或者 Reducer 可同时创建的最大的分区个数，默认为 100。

```sql
set hive.exec.max.dynamic.partitions.pernode=100
```

（5）一条 insert 语句可以创建的最大的文件个数，默认 100000。

```sql
set hive.exec.max.created.files=100000
```

（6）当查询结果为空时且进行动态分区时，是否抛出异常，默认 false。

```sql
hive.error.on.empty.partition=false
```

2）案例实操

需求：将 dept 表中的数据按照地区（ loc 字段），插入到目标表 dept_partition 的相应分区中。

（1）创建目标分区表

```sql
create table dept_partition_dynamic(
    id int, 
    name string
) 
partitioned by (loc int) 
row format delimited fields terminated by '\t';
```

（2）设置动态分区

```sql
set hive.exec.dynamic.partition.mode = nonstrict;
hive (default)> 
insert into table dept_partition_dynamic 
partition(loc) 
select 
    deptno, 
    dname, 
    loc 
from dept;
```

（3）查看目标分区表的分区情况

```sql
show partitions dept_partition_dynamic;
```

## 10.2 分桶表

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理的分区。对于一张表或者分区，Hive 可以进一步组织成桶，也就是更为细粒度的数据范围划分，分区针对的是数据的存储路径，分桶针对的是数据文件。

分桶表的基本原理是，首先为每行数据计算一个指定字段的数据的 hash 值，然后模以一个指定的分桶数，最后将取模运算结果相同的行，写入同一个文件中，这个文件就称为一个分桶（ bucket ）。

### 10.2.1 分桶表基本语法

1）建表语句

```sql
create table stu_buck(
    id int, 
    name string
)
clustered by(id) 
into 4 buckets
row format delimited fields terminated by '\t';
```

2）数据装载

（1）数据准备

在 /opt/module/hive/datas/ 路径上创建 student.txt 文件，并输入如下内容。

```txt title="student.txt"
1001	student1
1002	student2
1003	student3
1004	student4
1005	student5
1006	student6
1007	student7
1008	student8
1009	student9
1010	student10
1011	student11
1012	student12
1013	student13
1014	student14
1015	student15
1016	student16
```

（2）导入数据到分桶表中

!!! info "说明"

    Hive 新版本 load 数据可以直接跑 MapReduce，老版的 Hive 需要将数据传到一张表里，再通过查询的方式导入到分桶表里面。

```sql
load data local inpath '/opt/module/hive/datas/student.txt' 
into table stu_buck;
```

（3）查看创建的分桶表中是否分成 4 个桶

![image-20230216103452078](https://cos.gump.cloud/uPic/image-20230216103452078.png)

（4）观察每个分桶中的数据

### 10.2.2 分桶排序表

1）建表语句

```sql
create table stu_buck_sort(
    id int, 
    name string
)
clustered by(id) sorted by(id)
into 4 buckets
row format delimited fields terminated by '\t';
```

2）数据装载

（1）导入数据到分桶表中

```sql
load data local inpath '/opt/module/hive/datas/student.txt' 
into table stu_buck_sort;
```

（2）查看创建的分桶表中是否分成 4 个桶

![image-20230216103602339](https://cos.gump.cloud/uPic/image-20230216103602339.png)

（3）观察每个分桶中的数据

