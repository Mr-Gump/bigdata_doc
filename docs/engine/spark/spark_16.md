# 第16章 SparkSQL优化

Spark 3.0 大版本发布，Spark SQL 的优化占比将近 50%。Spark SQL 取代 Spark Core，成为新一代的引擎内核，所有其他子框架如Mllib、Streaming 和 Graph，都可以共享 Spark SQL 的性能优化，都能从 Spark 社区对于 Spark SQL 的投入中受益。

## 16.0 准备测试用表和数据

1）上传 3 个 log 到 hdfs 新建的 sparkdata 路径
2）hive 中创建 sparktuning 数据库
3）执行

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 4g  --class com.atguigu.sparktuning.utils.InitUtil spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar
```

## 16.1 执行计划

SparkSQL 具有和 Hive 语法类型的执行计划查询，使用语法如下：

```java
.explain(mode="xxx")
```

从 3.0 开始，explain 方法有一个新的参数 mode，该参数可以指定执行计划展示格式：

-  explain(mode="simple")：只展示物理执行计划。

- explain(mode="extended")：展示物理执行计划和逻辑执行计划。

- explain(mode="codegen") ：展示要Codegen生成的可执行Java代码。

- explain(mode="cost")：展示优化后的逻辑执行计划以及相关的统计。

- explain(mode="formatted")：以分隔的方式输出，它会输出更易读的物理执行计划，并展示每个节点的详细信息。

## 16.2 执行计划处理流程

![image-20230209112034871](https://cos.gump.cloud/uPic/image-20230209112034871.png)

核心的执行过程一共有5个步骤：

![image-20230209112048429](https://cos.gump.cloud/uPic/image-20230209112048429.png)

这些操作和计划都是 Spark SQL 自动处理的，会生成以下计划：

- Unresolved 逻辑执行计划：`== Parsed Logical Plan ==`
  Parser 组件检查 SQL 语法上是否有问题，然后生成 Unresolved（未决断）的逻辑计划，不检查表名、不检查列名
- Resolved 逻辑执行计划：`== Analyzed Logical Plan ==`
  通过访问 Spark 中的 Catalog 存储库来解析验证语义、列名、类型、表名等
- 优化后的逻辑执行计划：`== Optimized Logical Plan ==`
  Catalyst 优化器根据各种规则进行优化
- 物理执行计划：`== Physical Plan ==`

（1）HashAggregate 运算符表示数据聚合，一般 HashAggregate 是成对出现，第一个 HashAggregate 是将执行节点本地的数据进行局部聚合，另一个 HashAggregate 是将各个分区的数据进一步进行聚合计算

（2）Exchange 运算符其实就是 shuffle，表示需要在集群上移动数据。很多时候 HashAggregate 会以 Exchange 分隔开来

（3）Project 运算符是 SQL 中的投影操作，就是选择列（例如：select name, age…）

（4）BroadcastHashJoin 运算符表示通过基于广播方式进行 HashJoin

（5）LocalTableScan 运算符就是全表扫描本地的表

## 16.3 基于RBO的优化

SparkSQL 在整个执行计划处理的过程中，使用了 Catalyst 优化器。

在 Spark 3.0 版本中，Catalyst 总共有 81 条优化规则（Rules），分成 27 组（Batches），其中有些规则会被归类到多个分组里。因此，如果不考虑规则的重复性，27 组算下来总共会有 129 个优化规则。

如果从优化效果的角度出发，这些规则可以归纳到以下 3 个范畴：

### 16.3.1 谓词下推（Predicate Pushdown）

将过滤条件的谓词逻辑都尽可能提前执行，减少下游处理的数据量。对应 PushDownPredicte 优化规则，对于 Parquet、ORC 这类存储格式，结合文件注脚（Footer）中的统计信息，下推的谓词能够大幅减少数据扫描量，降低磁盘 I/O 开销。

左外关联下推规则：左表 left join 右表。

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 4 --executor-memory 6g  --class com.atguigu.sparktuning.rbo.PredicateTuning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

| 情况                | 左表       | 右表       |
| ------------------- | ---------- | ---------- |
| Join中条件（on）    | 只下推右表 | 只下推右表 |
| Join后条件（where） | 两表都下推 | 两表都下推 |

!!! warning "注意"

    外关联时，过滤条件写在 on 与 where，结果是不一样的！

### 16.3.2 列剪裁（Column Pruning）

列剪裁就是扫描数据源的时候，只读取那些与查询相关的字段。

### 16.3.3 常量替换（Constant Folding）

假设我们在年龄上加的过滤条件是 `age < 12 + 18`，Catalyst 会使用 ConstantFolding 规则，自动帮我们把条件变成 `age < 30`。再比如，我们在 select 语句中，掺杂了一些常量表达式，Catalyst 也会自动地用表达式的结果进行替换。

## 16.4 基于CBO的优化

CBO 优化主要在物理计划层面，原理是计算所有可能的物理计划的代价，并挑选出代价最小的物理执行计划。充分考虑了数据本身的特点（如大小、分布）以及操作算子的特点（中间结果集的分布及大小）及代价，从而更好的选择执行代价最小的物理执行计划。

而每个执行节点的代价，分为两个部分：

（1）该执行节点对数据集的影响，即该节点输出数据集的{++大小++}与{++分布++}

（2）该执行节点操作算子的代价

每个操作算子的代价相对固定，可用规则来描述。而执行节点输出数据集的大小与分布，分为两个部分：

（1）初始数据集，也即原始表，其数据集的大小与分布可直接通过统计得到
（2）中间节点输出数据集的大小与分布可由其{==输入数据集的信息与操作本身的特点推算==}

## 16.5 JOIN策略

Spark 提供了 5 种 JOIN 机制来执行具体的 JOIN 操作。选择的方式如下图：

![image-20230209112953829](https://cos.gump.cloud/uPic/image-20230209112953829.png)

### 16.5.1 影响Join的因素

#### 1）数据集的大小

参与 JOIN 的数据集的大小会直接影响 JOIN 操作的执行效率。同样，也会影响 JOIN 机制的选择和 JOIN 的执行效率。

#### 2）Join的条件

JOIN 的条件会涉及字段之间的逻辑比较。根据 JOIN 的条件，JOIN 可分为两大类：{++等值连接++}和{++非等值连接++}。等值连接会涉及一个或多个需要同时满足的相等条件。在两个输入数据集的属性之间应用每个等值条件。当使用其他运算符(运算连接符不为 `=` )时，称之为非等值连接。

### 3）Join的类型

在输入数据集的记录之间应用连接条件之后，JOIN 类型会影响 JOIN 操作的结果。主要有以下几种 JOIN 类型：

- {==内连接==}（Inner Join）：仅从输入数据集中输出匹配连接条件的记录
- {==外连接==}（Outer Join）：又分为左外连接、右外链接和全外连接
- 半连接（Semi Join）：右表只用于过滤左表的数据而不出现在结果集中
- 交叉连接（Cross Join）：交叉联接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉联接也称作笛卡尔积

### 16.5.2 Shuffle Hash Join

当执行数据量较大的等值连接时，使用 Shuffle Hash Join，操作流程为将两个表需要 Join 的 Key，进行 hash 重分区走 shuffle，写入磁盘之后，将相同分区编号的数据读取合并。

条件与特点：

（1）仅支持等值连接，join key 不需要排序
（2）支持除了全外连接( full outer joins )之外的所有 join 类型
（3）需要对{==小表构建 Hash map==}，属于{==内存密集型==}的操作，如果构建 Hash 表的一侧数据比较大，可能会造成 OOM
（4）将参数 `spark.sql.join.prefersortmergeJoin`  置为 false。

### 16.5.3 Broadcast Hash Join

广播 Join，也称为 Map 端 Join，该操作可以避免走 shuffle，是一种非常高效的 Join 操作。当一张表格比较小的时候，通常使用 Broadcast Hash Join。具体的操作为先将小表发送到 Driver 端，之后使用广播变量发送到每个 Executor 端。

1）通过参数指定自动广播

广播 join 默认值为 10MB，由 `spark.sql.autoBroadcastJoinThreshold` 参数控制。

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 4g  --class com.atguigu.sparktuning.join.AutoBroadcastJoinTuning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar
```

2）强行广播

修改使用的 sqlStr1 进行 join 提示。

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 4g  --class com.atguigu.sparktuning.join.AutoBroadcastJoinTuning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar
```

条件与特点

（1）仅支持等值连接，join key 不需要排序
（2）支持除了全外连接（ full outer joins ）之外的所有 join 类型
（3）Broadcast Hash Join 相比其他的 join 机制而言，效率更高。但是，Broadcast Hash Join 属于{==网络密集型==}的操作（数据冗余传输），除此之外，需要在 Driver 端缓存数据，所以当{==小表的数据量较大时，会出现 OOM 的情况==}
（4）被广播的小表的数据量要小于 `spark.sql.autoBroadcastJoinThreshold` 值，默认是 10MB (10485760)
（5）基表不能被 broadcast，比如左连接时，只能将右表进行广播。形如：fact_table.join(broadcast(dimension_table)，可以不使用 broadcast 提示，当满足条件时会自动转为该 join 方式。

### 16.5.4 Sort Merge Join

该 JOIN 机制是 Spark {==默认==}的，可以通过参数 `spark.sql.join.preferSortMergeJoin` 进行配置，默认是 {==true==}，即优先使用 Sort Merge Join。一般在两张大表进行 JOIN 时，使用该方式。

条件与特点：

（1）仅支持等值连接
（2）支持所有 join 类型
（3）Join Key 是排序的
（4）参数 `spark.sql.join.prefersortmergeJoin`   (默认 true )设定为 true

桶 join 优化：需要进行分桶，首先会进行排序，然后根据 key 值合并，把相同 key 的数据放到同一个 bucket 中（按照 key 进行 hash ）。分桶的目的其实就是把{++大表化成小表++}。相同 key 的数据都在同一个桶中之后，再进行 join 操作，那么在联合的时候就会大幅度的减小无关项的扫描。

使用条件：
（1）两表进行分桶，桶的个数必须相等
（2）两边进行 join 时，join 列 = 排序列 = 分桶列

不使用SMB Join：

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 6g  --class com.atguigu.sparktuning.join.BigJoinDemo spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar
```

使用SMB Join：

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 6g  --class com.atguigu.sparktuning.join.SMBJoinTuning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar
```

### 16.5.5 Cartesian Join

非等值连接时会使用 Cartesian Join。顾名思义，这种 Join 的结果就是得到两张表的笛卡尔积。

条件与特点：

（1）仅支持内连接
（2）支持等值和不等值连接
（3）开启参数 `spark.sql.crossJoin.enabled=true`

### 16.5.6 Broadcast Nested Loop Join

无法使用别的 Join 策略时的最后选择，一般在非等值连接，同时还不是内连接的时候使用。应该避免使用这种 Join 策略。

条件与特点：

（1）条件与特点
（2）支持等值和非等值连接
（3）支持所有的 JOIN 类型，主要优化点如下：

- 当右外连接时要广播左表
- 当左外连接时要广播右表
- 当内连接时，要广播左右两张表

## 16.6 AQE自适应优化

Spark 在 3.0 版本推出了 AQE（Adaptive Query Execution），即自适应查询执行。AQE 是 Spark SQL 的一种动态优化机制，在运行时，每当 Shuffle Map 阶段执行完毕，AQE 都会结合这个阶段的统计信息，基于既定的规则动态地调整、修正尚未执行的逻辑计划和物理计划，来完成对原始查询语句的运行时优化。

### 16.6.1 动态合并分区

在 Spark 中运行查询处理非常大的数据时，shuffle 通常会对查询性能产生非常重要的影响。shuffle 是非常昂贵的操作，因为它需要进行网络传输移动数据，以便下游进行计算。

最好的分区取决于数据，但是每个查询的阶段之间的数据大小可能相差很大，这使得该数字难以调整：

（1）如果分区太少，则每个分区的数据量可能会很大，处理这些数据量非常大的分区，可能需要将数据溢写到磁盘（例如，排序和聚合），降低了查询。

（2）如果分区太多，则每个分区的数据量大小可能很小，读取大量小的网络数据块，这也会导致 I/O 效率低而降低了查询速度。拥有大量的 task（一个分区一个 task ）也会给 Spark 任务计划程序带来更多负担。

 为了解决这个问题，我们可以在任务开始时先设置较多的 shuffle 分区个数，然后在运行时通过查看 shuffle 文件统计信息将相邻的小分区合并成更大的分区。

例如，假设正在运行 `select max(i) from tbl group by j`。输入 tbl 很小，在分组前只有 2 个分区。那么任务刚初始化时，我们将分区数设置为 5，如果没有 AQE，Spark 将启动五个任务来进行最终聚合，但是其中会有三个非常小的分区，为每个分区启动单独的任务这样就很浪费。

![image-20230209114442877](https://cos.gump.cloud/uPic/image-20230209114442877.png)

取而代之的是，AQE 将这三个小分区合并为一个，因此最终聚只需三个 task 而不是五个

![image-20230209114509806](https://cos.gump.cloud/uPic/image-20230209114509806.png)

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.AQEPartitionTunning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

结合动态申请资源：

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 2 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.DynamicAllocationTunning spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

### 16.6.2 动态切换Join策略

Spark 支持多种 join 策略，其中如果 join 的一张表可以很好的插入内存，那么 broadcast hash join 通常性能最高。因此，spark join 中，如果小表小于广播大小阀值（默认 10mb），Spark 将计划进行 broadcast hash join。但是，很多事情都会使这种大小估计出错（例如，存在选择性很高的过滤器），或者 join 关系是一系列的运算符而不是简单的扫描表操作。

为了解决此问题，AQE 现在根据最准确的 join 大小运行时重新计划 join 策略。从下图实例中可以看出，发现连接的右侧表比左侧表小的多，并且足够小可以进行广播，那么 AQE 会重新优化，将 sort merge join 转换成为 broadcast hash join。

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 4 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.AqeDynamicSwitchJoin spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

### 16.6.3 动态优化Join倾斜

当数据在群集中的分区之间分布不均匀时，就会发生数据倾斜。严重的倾斜会大大降低查询性能，尤其对于 join。AQE skew join 优化会从随机 shuffle 文件统计信息自动检测到这种倾斜。然后它将倾斜分区拆分成较小的子分区。

 例如，下图 A join B,A 表中分区 A0 明细大于其他分区。

![image-20230209114733021](https://cos.gump.cloud/uPic/image-20230209114733021.png)

因此，skew join 会将 A0 分区拆分成两个子分区，并且对应连接 B0 分区。

![image-20230209114748385](https://cos.gump.cloud/uPic/image-20230209114748385.png)

 没有这种优化，会导致其中一个分区特别耗时拖慢整个 stage,有了这个优化之后每个 task 耗时都会大致相同，从而总体上获得更好的性能。

（1）spark.sql.adaptive.skewJoin.enabled :是否开启倾斜 join 检测，如果开启了，那么会将倾斜的分区数据拆成多个分区,默认是{==开启==}的，但是得打开 aqe

（2）spark.sql.adaptive.skewJoin.skewedPartitionFactor :默认值 5，此参数用来判断分区数据量是否数据倾斜，当任务中最大数据量分区对应的数据量大于的分区中位数乘以此参数，并且也大于 `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` 参数，那么此任务是数据倾斜

（3）spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes :默认值 256mb，用于判断是否数据倾斜

（4）spark.sql.adaptive.advisoryPartitionSizeInBytes :此参数用来告诉 spark 进行拆分后推荐分区大小是多少

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 4 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.AqeOptimizingSkewJoin spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

如果同时开启了 `spark.sql.adaptive.coalescePartitions.enabled` 动态合并分区功能，那么会先合并分区，再去判断倾斜，将动态合并分区打开后，重新执行：

```shell
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 4 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.AqeOptimizingSkewJoin spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

修改中位数的倍数为 2，重新执行：

```java
spark-submit --master yarn --deploy-mode client --driver-memory 1g --num-executors 3 --executor-cores 4 --executor-memory 2g  --class com.atguigu.sparktuning.aqe.AqeOptimizingSkewJoin spark-tuning-1.0-SNAPSHOT-jar-with-dependencies.jar 
```

