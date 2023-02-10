# 第8章 Spark SQL概述

## 8.1 什么是Spark SQL

Spark SQL 是用于结构化数据处理的 Spark 模块。与基本的 Spark RDD API 不同，Spark SQL 提供的接口为 Spark 提供了有关数据结构和正在执行的计算的更多信息。在内部，Spark SQL 使用这些额外的信息来执行额外的优化。与 Spark SQL 交互的方式有多种，包括 SQL 和 Dataset API。计算结果时，使用相同的执行引擎，与您用于表达计算的 API / 语言无关。

## 8.2 为什么要有Spark SQL

## ![image-20230206184329489](https://cos.gump.cloud/uPic/image-20230206184329489.png)

## 8.3 SparkSQL的发展

1）发展历史

RDD（Spark1.0）=》Dataframe（Spark1.3）=》Dataset（Spark1.6）

如果同样的数据都给到这三个数据结构，它们分别计算之后，都会给出相同的结果。不同的是他们的{++执行效率++}和{++执行方式++}。在现在的版本中，DataSet 性能最好，已经成为了唯一使用的接口。其中 Dataframe 已经在底层被看做是特殊泛型的DataSet\<Row>。

2）三者的共性

- RDD、DataFrame、DataSet 全都是 Spark 平台下的分布式弹性数据集，为处理超大型数据提供便利
- 三者都有惰性机制，在进行创建、转换，如 map 方法时，不会立即执行，只有在{==遇到 Action 行动算子如 foreach 时，三者才会开始遍历运算==}
- 三者有许多{==共同的函数==}，如 filter，排序等
- 三者都会根据 Spark 的内存情况{==自动缓存运算==}
- 三者都有{==分区==}的概念

## 8.4 Spark SQL的特点

- [x] 易整合

无缝的整合了 SQL 查询和 Spark 编程。

![image-20230206184629893](https://cos.gump.cloud/uPic/image-20230206184629893.png)

- [x] 统一的数据访问方式

使用相同的方式连接不同的数据源。

![image-20230206184640853](https://cos.gump.cloud/uPic/image-20230206184640853.png)

- [x] 兼容 Hive

在已有的仓库上直接运行 SQL 或者 HQL。

![image-20230206184657337](https://cos.gump.cloud/uPic/image-20230206184657337.png)

- [x] 标准的数据连接

通过 JDBC 或者 ODBC 来连接。

![image-20230206184820802](https://cos.gump.cloud/uPic/image-20230206184820802.png)