# 第1章 Hive入门

## 1.1 什么是Hive

#### 1）Hive 简介

Hive 是由 Facebook 开源，基于 Hadoop 的一个{==数据仓库==}工具，可以{==将结构化的数据文件映射为一张表==}，并提供{==类 SQL==}  查询功能。

!!! question 

    那为什么会有 Hive 呢？它是为了解决什么问题而诞生的呢？

下面通过一个案例，来快速了解一下 Hive。

例如：需求，统计单词出现个数。

（1）在 Hadoop 课程中我们用 MapReduce 程序实现的，当时需要写 Mapper、Reducer 和 Driver 三个类，并实现对应逻辑，相对繁琐。

```text
test表
id列

atguigu
atguigu
ss
ss
jiao
banzhang
xue
hadoop
```

（2）如果通过 Hive SQL 实现，一行就搞定了，简单方便，容易理解。

```sql
select count(*) from test group by id;
```

#### 2）Hive 本质

Hive 是一个 Hadoop 客户端，用于{==将 HQL（Hive SQL）转化成 MapReduce 程序==}。

（1）Hive 处理的数据存储在 HDFS

（2）Hive 分析数据底层的实现是 MapReduce（也可配置为 Spark 或者 Tez） 

（3）执行程序运行在 Yarn 上

## 1.2 Hive架构原理

![image-20230212203525941](https://cos.gump.cloud/uPic/image-20230212203525941.png)

1）用户接口：Client

CLI（ command-line interface ）、JDBC / ODBC。

!!! info "说明"

    JDBC 和 ODBC 的区别

        （1）JDBC 的移植性比 ODBC 好；（通常情况下，安装完 ODBC 驱动程序之后，还需要经过确定的配置才能够应用。而不相同的配置在不相同数据库服务器之间不能够通用。所以，安装一次就需要再配置一次。JDBC 只需要选取适当的 JDBC 数据库驱动程序，就不需要额外的配置。在安装过程中，JDBC 数据库驱动程序会自己完成有关的配置。）

        （2）两者使用的语言不同，JDBC 在 Java 编程时使用，ODBC 一般在 C / C++ 编程时使用。

2）元数据：Metastore

元数据包括：数据库（默认是 default ）、表名、表的拥有者、列/分区字段、表的类型（是否是外部表）、表的数据所在目录等。

默认存储在自带的 {==derby==} 数据库中，由于 derby 数据库只支持单客户端访问，生产环境中为了多人开发，推荐使用 MySQL 存储 Metastore。

3）驱动器：Driver

- 解析器（SQLParser）：将 SQL 字符串转换成抽象语法树（ AST ）

- 逻辑计划生成器（ Logical Plan Gen ）：将语法树生成逻辑计划

- 逻辑优化器（ Logical Optimizer ）：对逻辑计划进行优化
- 物理计划生成器（ Physical Plan Gen ）：根据优化后的逻辑计划生成物理计划
- 物理优化器（ Physical Optimizer ）：对物理计划进行优化
- 执行器（ Execution ）：执行该计划，得到查询结果并返回给客户端

<figure markdown>
  ![image-20230212204016276](https://cos.gump.cloud/uPic/image-20230212204016276.png)
  <figcaption>抽象语法树</figcaption>
</figure>

<figure markdown>
  ![image-20230212204037345](https://cos.gump.cloud/uPic/image-20230212204037345.png)
  <figcaption>逻辑执行计划和物理执行计划</figcaption>
</figure>
4）Hadoop

使用 HDFS 进行存储，可以选择 MapReduce / Tez / Spark 进行计算。

## 1.3 Hive的优缺点

1）优点

（1）操作接口采用类 SQL 语法，提供快速开发的能力（简单、容易上手）。

（2）避免了去写 MapReduce，减少开发人员的学习成本。

2）缺点

（1）Hive 的执行延迟比较高，因此 Hive 常用于数据分析，对{==实时性要求不高==}的场合。

（2）Hive 分析的数据是存储在 HDFS 上，HDFS 不支持随机写，只支持追加写，所以在 {==Hive 中不能高效 update 和 delete==}。

## 1.4 Hive和数据库比较 

| 对比项   | Hive                    | MySQL                            |
| -------- | ----------------------- | -------------------------------- |
| 数据规模 | 大数据 pb 及以上        | 数据量小一般百万左右到达单表极限 |
| 数据存储 | HDFS                    | 拥有自己的存储引擎               |
| 计算引擎 | MapReduce / Spark / Tez | 自己的引擎 InnoDB                |
| 执行延迟 | 高                      | 低                               |

综上能发现，Hive 压根不是数据库，Hive 除了语言类似以外，存储和计算都是用的 Hadoop 的，但是 MySQL 却用的是自己的，拥有自己的体系。



