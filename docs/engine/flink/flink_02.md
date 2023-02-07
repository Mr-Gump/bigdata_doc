# 第二章 Flink运行时架构

## 2.1 Flink主从架构

- Flink 运行时架构由两种类型的进程组成
  - 一个 JobManager（作业管理器，Master 进程）
  - 一个或者多个 TaskManager（任务管理器，Slave 进程）

- 典型的Master-Slave（主从）架构

![23](https://cos.gump.cloud/uPic/23.svg)


## 2.2 作业管理器

作业管理器是一个 JVM 进程。进程中包含三类线程：

- Flink 的资源管理器（ResourceManager）：资源是任务管理器中的任务插槽（Task Slot）
- 分发器（WebUI）：提交任务和监控集群和任务
- JobMaster（作业主线程，每个作业对应一个）：调度任务，将 DAG 部署到任务管理器。
  

![24](https://cos.gump.cloud/uPic/24.svg)

JobMaster 由于是每个作业对应一个，所以可能有多个 JobMaster 线程。

## 2.3 任务管理器

- 任务管理器也是一个 JVM 进程。包含至少一个任务插槽
- 任务插槽是 Flink 的最小计算单元
- 每个任务插槽是一个内存分片，每个任务插槽占用一段内存
- 一个任务插槽中至少运行一个线程
- 任务插槽内存大小 = 任务管理器的 JVM 堆内存 / 任务插槽数量

## 2.4 任务插槽

- 不同算子的并行子任务可以共享同一个任务插槽
- 相同算子的不同并行子任务不能共享同一个任务插槽
- 算子的并行度是 N（`.map().setParallelism(N)`），那么算子就有 N 个并行子任务，并且必须占用 N 个任务插槽。

给定以下程序以及 6 个任务插槽，那么可能的任务调度情形如下图：

```java
source.setParallelism(2)
  .map.setParallelism(2)
  .keyBy.window
  .reduce.setParallelism(2)
  .sink.setParallelism(1)
```

![tasks_slots](https://cos.gump.cloud/uPic/tasks_slots.svg)

给定以下程序以及 6 个任务插槽，可能的调度情形如下图：

```java
source.setParallelism(6)
  .map.setParallelism(6)
  .keyBy.window
  .reduce.setParallelism(6)
  .sink.setParallelism(1)
```

![slot_sharing](https://cos.gump.cloud/uPic/slot_sharing.svg)

![slots_parallelism](https://cos.gump.cloud/uPic/slots_parallelism.svg)

## 2.5 并行度的设置

并行度指的是算子的并行度。

算子并行度的大小不能超过集群可用任务插槽的数量。

从上到下，优先级升高。

1. 任务管理器的配置文件里面：`flink-conf.yaml`中的配置项`parallelism.default: 1`
2. 在命令行或者 WebUI 提交任务时指定并行度：`./bin/flink run jar包 -p 16`
3. 全局并行度：`env.setParallelism(1)`
4. 针对算子设置并行度：`.print().setParallelism(1)`

## 2.6 并行度设置的最佳实践

1. 不要设置全局并行度，因为没办法在命令行或者 WebUI 提交任务时做动态扩容。
2. 针对某些算子设置并行度，例如数据源，为了不改变数据的顺序，设置数据源的并行度为 1。
3. 在提交任务时设置，可以动态扩容。

```java
env
  .source.setParallelism(1)
  .keyBy
  .reduce // 不针对reduce设置并行度
  .print.setParallelism(1)
```

第一次提交任务

```bash
$ flink run jar包 -p 8
```

那么 `reduce` 算子的并行度是 8，`source` 和 `print` 的并行度是 1。

动态扩容

```bash
$ flink run jar包 -p 16
```

那么 `reduce` 算子的并行度是 16，`source` 和 `print` 的并行度是 1。

## 2.7 任务提交流程

独立集群的任务提交流程。以下示意图 1 个作业管理器。3 个任务管理器。共占用 4 台机器。

![13](https://cos.gump.cloud/uPic/13.svg)

- 在集群启动时，任务管理器会向资源管理器注册自己的任务插槽
- 任务管理器之间存在数据的交换

## Flink中的DAG（有向无环图）数据结构

![9](https://cos.gump.cloud/uPic/9.svg)

- StreamGraph：是根据用户通过 Stream API 编写的代码生成的最初的有向无环图。用来表示程序的拓扑结构。源码：`StreamGraph.java`
- JobGraph：StreamGraph 在编译的阶段经过优化后生成了 JobGraph（源码：`JobGraph.java`），JobGraph 就是提交给作业管理器的数据结构。主要的优化为，将多个符合条件（没有 shuffle，并行度相同）的算子串在一起作为一个{==算子链节点==}。保证同一个算子链节点里面的所有算子都在同一个任务插槽的同一个线程中执行。这样算子之间的数据就是本地转发（无需序列化反序列化和网络 IO）。两个条件：
  - 两个算子之间没有 shuffle 存在
  - 两个算子的并行度必须相同
- ExecutionGraph：作业管理器根据 JobGraph 生成 ExecutionGraph（源码：`ExecutionGraph.java`）。ExecutionGraph 是 JobGraph 的并行化版本，是调度层最核心的数据结构。
- 物理执行图：JobMaster 线程根据 ExecutionGraph 对作业进行调度后，在各个任务管理器上部署 Task 后形成的“图”，并不是一个具体的数据结构。

我们看一下如下伪代码的 DAG 是如何进行转换的。

```java
source.setParallelism(1)
  .flatMap().setParallelism(2)
  .keyBy()
  .reduce().setParallelism(2)
  .sink().setParallelism(2)
```

首先生成的是 StreamGraph。

![10](https://cos.gump.cloud/uPic/10.svg)

StreamGraph 在客户端编译时生成了 JobGraph。

- source 和 flatMap 由于并行度不同，所以无法合并成一个算子链
- flatMap 和 reduce 虽然并行度相同，但由于算子之间存在 shuffle 所以也无法合并成一个算子链
- reduce 和 sink 并行度相同，且不存在 shuffle，所以可以合成一个算子链
  

![11](https://cos.gump.cloud/uPic/11.svg)

将 JobGraph 提交到作业管理器，会生成 ExecutionGraph，也就是将算子按照并行度拆分成多个并行子任务。

ExecutionGraph 中的每个顶点都要占用一个线程。所以下图中共有 5 个顶点，需要 5 个线程来执行。每个顶点（线程）对应了源码中的一个 `Task.java` 的实例。

![12](https://cos.gump.cloud/uPic/12.svg)

上图会占用几个任务插槽？

最少需要占用 2 个任务插槽。

- 第一个任务插槽：`source[1]`，`flatMap[1/2]`，`reduce->sink[1/2]`
- 第二个任务插槽：`flatMap[2/2]`，`reduce->sink[2/2]`

最多占用 5 个任务插槽。也就是 5 个顶点的线程分别占用一个任务插槽。

JobMaster 将 ExecutionGraph 部署到任务管理器执行。