# 第1章 Spark概述

## 1.1 什么是Spark

回顾：Hadoop 主要解决海量数据的{++存储++}和海量数据的{++分析计算++}。

Spark 是一种基于{==内存==}的快速、通用、可扩展的大数据{++分析计算引擎++}。

## 1.2 Hadoop与Spark历史

=== "Hadoop历史"

    ![image-20230130200801210](https://cos.gump.cloud/uPic/image-20230130200801210.png)

=== "Spark历史"

    ![image-20230130200844185](https://cos.gump.cloud/uPic/image-20230130200844185.png)

Hadoop 的 Yarn 框架比 Spark 框架诞生的晚，所以 Spark 自己也设计了一套资源调度框架。

## 1.3 Hadoop与Spark框架对比
=== "Hadoop MR 框架"

    ![image-20230130201115991](https://cos.gump.cloud/uPic/image-20230130201115991.png)

=== "Spark 框架"

    ![image-20230130201140660](https://cos.gump.cloud/uPic/image-20230130201140660.png)

## 1.4 Spark内置模块

![image-20230130201322438](https://cos.gump.cloud/uPic/image-20230130201322438.png)

=== "Spark Core"

    实现了 Spark 的基本功能，包含任务调度、内存管理、错误恢复、与存储系统交互等模块。Spark Core 中还包含了对{++弹性分布式数据集++}( Resilient Distributed DataSet，简称 RDD )的 API 定义。 



=== "Spark SQL"

    是 Spark 用来操作结构化数据的程序包。通过 Spark SQL，我们可以使用 SQL 或者 Apache Hive 版本的 HQL 来查询数据。Spark SQL 支持多种数据源，比如 Hive 表、Parquet 以及 JSON 等。



=== "Spark Streaming"

    是 Spark 提供的对实时数据进行流式计算的组件。提供了用来操作数据流的 API，并且与 Spark Core 中的 RDD API 高度对应。



=== "Spark MLlib"

    提供常见的机器学习功能的程序库。包括分类、回归、聚类、协同过滤等，还提供了模型评估、数据 导入等额外的支持功能。



=== "Spark GraphX"

    主要用于图形并行计算和图挖掘系统的组件。



=== "集群管理器"

    Spark 设计为可以高效地在一个计算节点到数千个计算节点之间伸缩计算。为了实现这样的要求，同时获得最大灵活性，Spark 支持在各种集群管理器（Cluster Manager）上运行，包括 Hadoop YARN、Apache Mesos，以及 Spark 自带的一个简易调度器，叫作{++独立调度器++}。

Spark 得到了众多大数据公司的支持，这些公司包括 Hortonworks、IBM、Intel、Cloudera、MapR、Pivotal、百度、阿里、腾讯、京东、携程、优酷土豆。当前百度的 Spark 已应用于大搜索、直达号、百度大数据等业务；阿里利用 GraphX 构建了大规模的图计算和图挖掘系统，实现了很多生产系统的推荐算法；腾讯 Spark 集群达到 8000 台的规模，是当前已知的世界上最大的 Spark 集群。

## 1.5 Spark特点
- 快：与 Hadoop 的 MapReduce 相比，Spark 基于{++内存++}的运算要快 {==100 倍以上==}，基于{++硬盘++}的运算也要快 {==10 倍以上==}。Spark 实现了高效的 DAG 执行引擎，可以通过基于内存来高效处理数据流。计算的中间结果是存在于内存中的。
- 易用：Spark 支持 {== Java、Python 和 Scala 的 API==}，还支持超过 80 种高级算法，使用户可以快速构建不同的应用。而且 Spark 支持交互式的 Python 和 Scala 的 Shell，可以非常方便地在这些 Shell 中使用 Spark 集群来验证解决问题的方法。
- 通用：Spark 提供了统一的解决方案。Spark 可以用于，交互式查询（{++Spark SQL++}）、实时流处理（{++Spark Streaming++}）、机器学习（{++Spark MLlib++}）和图计算（{++GraphX++}）。这些不同类型的处理都可以在同一个应用中无缝使用。减少了开发和维护的人力成本和部署平台的物力成本。
- 兼容性：Spark 可以非常方便地与其他的开源产品进行融合。比如，Spark {==可以使用 Hadoop 的 YARN==} 和 Apache Mesos 作为它的资源管理和调度器，并且可以处理{==所有 Hadoop 支持的数据==}，包括 HDFS、HBase 等。这对于已经部署 Hadoop 集群的用户特别重要，因为不需要做任何数据迁移就可以使用 Spark 的强大处理能力。



