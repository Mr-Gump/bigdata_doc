# 第5章 DML（Data Manipulation Language）数据操作

## 5.1 数据导入

### 5.1.1 向表中装载数据（Load）

**1）语法**

```sql
load data [local] inpath '数据的path' 
[overwrite] into table table_name [partition (partcol1=val1,…)];
```

（1）load data：表示加载数据。

（2）local：表示从本地加载数据到 Hive 表；否则从 HDFS 加载数据到 Hive 表。

（3）inpath：表示加载数据的路径。

（4）overwrite：表示覆盖表中已有数据，否则表示追加。

（5）into table：表示加载到哪张表。

（6）student：表示具体的表。

（7）partition：表示上传到指定分区。

**2）实操案例**

（0）创建一张表

```sql
create table student(
    id int, 
    name string
) 
row format delimited fields terminated by '\t';
```

（1）加载本地文件到 hive

```sql
load data local inpath '/opt/module/hive/datas/student.txt' into table student;
```

（2）加载 HDFS 文件到 hive 中

①上传文件到 HDFS

```shell
dfs -put /opt/module/hive/datas/student.txt /user/atguigu;
```

②加载 HDFS 上数据，导入完成后去 HDFS 上查看文件是否还存在

```sql
load data inpath '/user/atguigu/student.txt' into table student;
```

（3）加载数据覆盖表中已有的数据

①上传文件到 HDFS

```shell
dfs -put /opt/module/hive/datas/student.txt /user/atguigu;
```

②加载数据覆盖表中已有的数据

```sql
load data inpath '/user/atguigu/student.txt' overwrite into table student;
```

### 5.1.2 通过查询语句向表中插入数据（Insert）

**1）创建一张表**

```sql
create table student3(
    id int, 
    name string
) 
row format delimited fields terminated by '\t';
```

**2）基本模式插入数据**

```sql
insert into table  student3 values(1,'wangwu'),(2,'zhaoliu');
```

**3）根据查询结果插入数据**

```sql
insert overwrite table student3 
select 
    id, 
    name 
from student 
where id < 1006;
```

insert into：以追加数据的方式插入到表或分区，原有数据不会删除。

insert overwrite：会覆盖表中已存在的数据。

!!! info "注意"

    insert 不支持插入部分字段，并且后边跟 select 语句时，select 之前不能加 as，加了 as 会报错，一定要跟下面的 as select 区分开。

### 5.1.3 查询语句中创建表并加载数据（As Select）

根据查询结果创建表（查询的结果会添加到新创建的表中）。

```sql
create table if not exists student4 as select id, name from student;
```

### 5.1.4 创建表时通过Location指定加载数据路径

**1）上传数据到 HDFS 上**

```shell
hadoop fs -mkdir -p /student5;
hadoop fs -put student.txt /student5
```

**2）创建表，并指定在 HDFS 上的位置**

```sql
create external table if not exists student5(
    id int, 
    name string
)
row format delimited fields terminated by '\t'
location '/student5';
```

**3）查询数据**

```sql
select * from student5;
```

### 5.1.5 Import数据到指定Hive表中

!!! info "注意"

    先用 export 导出后，再将数据导入。并且因为 export 导出的数据里面{==包含了元数据==}，因此 import 要导入的表不可以存在，否则报错。

```sql
import table student2 from '/user/hive/warehouse/export/student';
```

## 5.2 数据导出

### 5.2.1 Insert导出

**1）将查询的结果导出到本地**

```sql
insert overwrite local directory '/opt/module/hive/datas/export/student' 
select * from student;
```

**2）将查询的结果格式化导出到本地**

```sql
insert overwrite local directory '/opt/module/hive/datas/export/student' 
row format delimited fields terminated by '\t' 
select * from student;
```

**3）将查询的结果导出到 HDFS 上（没有 local ）**

```sql
insert overwrite directory '/user/atguigu/student2' 
row format delimited fields terminated by '\t' 
select * from student;
```

!!! info "注意"

    insert 导出，导出的目录不用自己提前创建，Hive 会帮我们自动创建，但是由于是 overwrite，所以导出路径一定要写具体，否则很可能会误删数据。这个步骤很重要，切勿大意。

### 5.2.2 Export导出到HDFS

```sql
export table default.student to 
 '/user/hive/warehouse/export/student';
```

Export 和 Import 主要用于两个 Hadoop 平台集群之间 Hive 表迁移，不能直接导出到本地。

