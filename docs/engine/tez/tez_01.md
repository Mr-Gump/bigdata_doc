# 第1章 Tez概述 

## 1.1 Tez简介

Apache Tez 是 Hadoop 生态中的一个高性能批处理计算框架。其突出特点是使用 DAG（有向无环图）来描述计算任务，因此在复杂计算的场景下，Tez 能提供比 Map Reduce 更好的性能，下图是 Map Reduce 和 Tez 编程模型的对比。

![image-20230206221618169](https://cos.gump.cloud/uPic/image-20230206221618169.png)

## 1.2 Tez架构

Tez 基于 Hadoop Yarn 进行构建，它使用 Yarn 管理集群和分配资源。一个 Tez 的分布式计算程序包括一个 Tez Application Master 和若干个 Tez Task。Tez Application Master 负责整个 Tez 计算程序的调度执行以及计算资源的申请与分配。Tez Task 则负责执行具体的计算逻辑。Tez Application Master 和Tez Task 均运行在 Yarn 的 Container 中。

![image-20230206221752838](https://cos.gump.cloud/uPic/image-20230206221752838.png)

## 1.3 Tez术语

1）DAG（Direct Acyclic Graph）

有向无环图，Tez 使用有向无环图描述计算任务。

2）Vertex

顶点，表示一个数据的转换逻辑。

3）Edge

边，表示两个顶点之间的数据移动。

![image-20230206221855126](https://cos.gump.cloud/uPic/image-20230206221855126.png)