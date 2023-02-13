# 第4章 DDL（Data Definition Language）数据定义

## 4.1 创建数据库

**1）语法**

```sql
CREATE DATABASE [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

**2）创建一个数据库，数据库在 HDFS 上的默认存储路径是 /user/hive/warehouse/*.db。**

```sql
create database db_hive;
```

**3）避免要创建的数据库已经存在错误，增加 if not exists 判断。（标准写法）**

```sql
create database if not exists db_hive;
```

**4）创建一个数据库，指定数据库在 HDFS 上存放的位置**

```sql
create database db_hive2 
location '/db_hive2';
```

## 4.2 查询数据库

### 4.2.1 显示数据库

**1）显示数据库**

```sql
show databases;
```

**2）过滤显示查询的数据库**

```sql
show databases like 'db_hive*';
```

### 4.2.2 查看数据库详情

**1）显示数据库信息**

```sql
desc database db_hive;
```

**2）显示数据库详细信息，extended**

```sql
desc database extended db_hive;
```

### 4.2.3 切换当前数据库

```sql
use db_hive;
```

## 4.3 修改数据库

用户可以使用 alter database 命令为某个数据库的 dbproperties 设置键-值对属性值，来描述这个数据库的属性信息。{==数据库的其他元数据信息都是不可更改的，包括数据库名和数据库所在的目录位置==}。

```sql
alter database db_hive set dbproperties('createtime'='20220830');
```

1）查看 Hive 中基本信息。

```sql
desc database db_hive;

db_name	comment	location	owner_name	owner_type	parameters
db_hive		hdfs://hadoop102:8020/user/hive/warehouse/db_hive.db	atguigu	USER
```

2）查看 Hive 中详细信息。

```sql
desc database extended db_hive;
db_name comment location        owner_name      owner_type      parameters
db_hive         hdfs://hadoop102:8020/user/hive/warehouse/db_hive.db    atguigu USER    {createtime=20170830}
```

## 4.4 删除数据库

**1）删除空数据库**

```sql
drop database db_hive2;
```

**2）如果删除的数据库不存在，最好采用 if exists 判断数据库是否存在**

```sql
drop database if exists db_hive2;
```

**3）如果数据库不为空，可以采用 cascade 命令，强制删除**

```sql
drop database db_hive cascade;
```

## 4.5 创建表

1）建表语法

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
[LIKE table_name]
```

**2）字段解释说明** 

1）create table 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 if not exists 选项来忽略这个异常。

（2）external 关键字可以让用户创建一个外部表，在建表的同时可以指定一个指向实际数据的路径（ location ），{==在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据==}。

（3）comment：为表和列添加注释。

（4）partitioned by 创建分区表。

（5）clustered by 创建分桶表。

（6）sorted by 不常用，对桶中的一个或多个列另外排序。

（7）row format delimited

```sql
[FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char]
        [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 
   | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]
```

用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 row format 或者 row format delimited，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，{==Hive 通过 SerDe 确定表的具体的列的数据==}。

SerDe 是 Serialize / Deserilize 的简称，Hive 使用 Serde 进行行对象的序列与反序列化。

（8）stored as 指定存储文件类型

常用的存储文件类型：sequencefile（二进制序列文件）、textfile（文本）、rcfile（列式存储格式文件）。

如果文件数据是纯文本，可以使用 stored as textfile。如果数据需要压缩，使用 stored as sequencefile。

（9）location：指定表在 HDFS 上的存储位置。

（10）as：后跟查询语句，根据查询结果创建表。

（11）like：允许用户复制现有的表结构，但是不复制数据。

### 4.5.1 管理表

**1）理论**

默认创建的表都是所谓的{==管理表==}，有时也被称为{==内部表==}。因为这种表，Hive 会（或多或少地）控制着数据的生命周期。Hive 默认情况下会将这些表的数据存储在由配置项 `hive.metastore.warehouse.dir`（{==默认路径，/user/hive/warehouse==}）所定义的目录的子目录下。{==当我们删除一个管理表时，Hive 也会删除这个表中数据==}。管理表不适合和其他工具共享数据。

**2）案例实操**

（0）原始数据

在 /opt/module/hive/datas 路径上，创建 student.txt 文件，并输入如下内容。

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

（1）普通创建表

```sql
hive (default)>
create table if not exists student(
    id int, 
    name string
)
row format delimited fields terminated by '\t'
location '/user/hive/warehouse/student';
```

上传数据

```shell
hadoop fs -put student.txt /user/hive/warehouse/student
```

（2）根据查询结果创建表（查询的结果会添加到新创建的表中）

```sql
create table if not exists student2 as select id, name from student;
```

（3）根据已经存在的表结构创建表

```sql
create table if not exists student3 like student;
```

（4）查询表的类型

```sql
desc formatted student2;
```

（5）删除管理表，并查看表数据是否还存在

```sql
drop table student2;
```

### 4.5.2 外部表

**1）理论**

因为表是外部表，所以 Hive 并非认为其完全拥有这份数据。删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

**2）管理表和外部表的使用场景**

每天将收集到的网站日志定期流入 HDFS 文本文件。在外部表（原始日志表）的基础上做大量的统计分析，用到的中间表、结果表使用内部表存储，数据通过 select + insert 进入内部表。

**3）案例实操**

创建老师表，并向表中导入数据。

（0）原始数据

在 /opt/module/hive/datas 路径上，创建 teacher.txt 文件，并输入如下内容。

```txt title="teacher.txt"
1001	teacher1
1002	teacher2
1003	teacher3
1004	teacher4
1005	teacher5
1006	teacher6
1007	teacher7
1008	teacher8
1009	teacher9
1010	teacher10
1011	teacher11
1012	teacher12
1013	teacher13
1014	teacher14
1015	teacher15
1016	teacher16
```

（1）上传数据到 HDFS

```shell
hadoop fs -mkdir -p /school/teacher

hadoop fs -put teacher.txt /school/teacher
```

（2）建表语句，创建外部表

```sql
create external table if not exists teacher(
    id int, 
    name string
)
row format delimited fields terminated by '\t'
location '/school/teacher';
```

（3）查看创建的表

```sql
show tables;
```

（4）查看表格式化数据

```sql
desc formatted teacher;
```

（5）删除外部表

```sql
drop table teacher;
```

外部表删除后，HDFS 中的数据还在，但是 metadata 中 teacher 的元数据已被删除。

![image-20230213191906701](https://cos.gump.cloud/uPic/image-20230213191906701.png)

### 4.5.3 管理表与外部表的互相转换

（1）查询表的类型

```sql
desc formatted student3;
```

（2）修改内部表 student 为外部表

```sql
alter table student3 set tblproperties('EXTERNAL'='TRUE');
```

（3）查询表的类型

```sql
desc formatted student3;
```

（4）修改外部表 student 为内部表

```sql
alter table student3 set tblproperties('EXTERNAL'='FALSE');
```

（5）查询表的类型

```sql
desc formatted student3;
```

注意：`('EXTERNAL'='TRUE')`和`('EXTERNAL'='FALSE')` 为固定写法，{==区分大小写==}。

## 4.6 修改表

### 4.6.1 重命名表

**1）语法**

```sql
ALTER TABLE table_name RENAME TO new_table_name
```

**2）实操案例**

```sql
alter table student3 rename to student2;
```

### 4.6.2 增加/修改/替换列信息

**1）语法**

（1）更新列

更新列，列名可以随意修改，列的类型只能小改大，{==不能大改小==}（遵循自动转换规则）。

```sql
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment]
```

（2）增加和替换列

```sql
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) 
```

!!! info "注"

    ADD 是代表新增一个字段，字段位置在所有列后面（ partition 列前），REPLACE 则是表示替换表中所有字段，REPLACE 使用的时候，字段的类型要跟之前的类型对应上，数量可以减少或者增加，其实就是包含了更新列，增加列，删除列的功能。

**2）实操案例**

（1）查询表结构

```sql
desc student;
```

（2）添加列

```sql
alter table student add columns(age int);
```

（3）查询表结构

```sql
desc student;
```

（4）更新列

```sql
alter table student change column age ages double;
```

（5）查询表结构

```sql
desc student;
```

（6）替换列

```sql
alter table student replace columns(id int, name string);
```

（7）查询表结构

```sql
desc student;
```

## 4.7 删除表

```sql
drop table student2;
```

## 4.8 清除表

注意：truncate 只能删除管理表，{==不能删除外部表中数据==}。

```sql
truncate table student;
```



