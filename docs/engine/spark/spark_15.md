# 第15章 Spark内存管理

## 15.1 堆内和堆外内存

### 15.1.1 概念

Spark 支持堆内内存也支持堆外内存。

（1）堆内内存：程序在运行时动态地申请某个大小的内存空间

（2）堆外内存：直接向操作系统进行申请的内存，不受 JVM 控制

![image-20230208194839483](https://cos.gump.cloud/uPic/image-20230208194839483.png)

### 15.1.2 堆内内存和对外内存优缺点

1）堆外内存，相比于堆内内存有几个优势

（1）减少了垃圾回收的工作，因为垃圾回收会暂停其他的工作

（2）加快了复制的速度。因为堆内在 Flush 到远程时，会先序列化，然后再发送；而堆外内存本身是序列化的相当于省略掉了这个工作。

说明：{==堆外内存是序列化的==}，其占用的内存大小可直接计算。{==堆内内存是非序列化的对象==}，其占用的内存是通过{==周期性地采样近似估算而得==}。此外，在被 Spark 标记为释放的对象实例，很有可能在实际上并没有被 JVM 回收，导致实际可用的内存小于 Spark 记录的可用内存。所以 {==Spark 并不能准确记录实际可用的堆内内存==}，从而也就{==无法完全避免内存溢出 OOM 的异常==}。 

2）堆外内存，相比于堆内内存有几个缺点： 

（1）堆外内存难以控制，如果内存泄漏，那么很难排查 

（2）堆外内存相对来说，不适合存储很复杂的对象。一般简单的对象或者扁平化的比较适合。

### 15.1.3 如何配置

1）堆内内存大小设置：`-executor-memory` 或 `spark.executor.memory`

2）在默认情况下堆外内存并不启用，`spark.memory.offHeap.enabled` 参数启用，并由 `spark.memory.offHeap.size` 参数设定堆外空间的大小。

官网配置地址：http://spark.apache.org/docs/3.0.0/configuration.html

## 15.2 堆内内存空间分配

堆内内存包括：{++存储（Storage）内存++}、{++执行（Execution）内存++}、{++其他内存++}

### 15.2.1 静态内存管理

在 Spark 最初采用的静态内存管理机制下，存储内存、执行内存和其他内存的大小在 Spark 应用程序运行期间均为固定的，但用户可以应用程序启动前进行配置，堆内内存的分配如图所示：

![image-20230208195316592](https://cos.gump.cloud/uPic/image-20230208195316592.png)

可以看到，可用的堆内内存的大小需要按照下列方式计算：

```bash
可用的存储内存 = systemMaxMemory * spark.storage.memoryFraction * spark.storage.safety Fraction
可用的执行内存 = systemMaxMemory * spark.shuffle.memoryFraction * spark.shuffle.safety Fraction
```

![image-20230208195408347](https://cos.gump.cloud/uPic/image-20230208195408347.png)

{==由于新的内存管理机制的出现，这种方式目前已经很少有开发者使用==}，出于兼容旧版本的应用程序的目的，Spark 仍然保留了它的实现。

### 15.2.2 统一内存管理

{==Spark1.6==} 之后引入的统一内存管理机制，与静态内存管理的区别在于存储内存和执行内存共享同一块空间，可以动态占用对方的空闲区域，统一内存管理的堆内内存结构如图所示：

![image-20230208195444364](https://cos.gump.cloud/uPic/image-20230208195444364.png)

统一内存管理的堆外内存结构如下图所示：

![image-20230208195457160](https://cos.gump.cloud/uPic/image-20230208195457160.png)

其中最重要的优化在于动态占用机制，其规则如下：

（1）设定基本的存储内存和执行内存区域（spark.storage.storageFraction 参数），该设定确定了双方各自拥有的空间的范围

（2）双方的空间都不足时，则存储到硬盘；若己方空间不足而对方空余时，可借用对方的空间;（存储空间不足是指不足以放下一个完整的 Block）

（3）执行内存的空间被对方占用后，可让对方将占用的部分转存到硬盘，然后”归还”借用的空间

（4）存储内存的空间被对方占用后，无法让对方”归还”，因为需要考虑 Shuffle 过程中的很多因素，实现起来较为复杂

统一内存管理的动态占用机制如图所示：

![image-20230208195626768](https://cos.gump.cloud/uPic/image-20230208195626768.png)

## 15.3 存储内存管理

### 15.3.1 RDD的持久化机制

RDD 的持久化由 Spark 的 Storage 模块负责，实现了 RDD 与物理存储的解耦合。Storage 模块负责管理 Spark 在计算过程中产生的数据，将那些在内存或磁盘、在本地或远程存取数据的功能封装了起来。在具体实现时 Driver 端和 Executor 端的 Storage 模块构成了主从式的架构，即 Driver 端的 BlockManager 为 Master，Executor 端的 BlockManager 为 Slave。

Storage 模块在逻辑上以 Block 为基本存储单位，RDD 的每个 Partition 经过处理后唯一对应一个 Block（BlockId 的格式为 rdd_RDD-ID_PARTITION-ID ）。Driver 端的 Master 负责整个 Spark 应用程序的 Block 的元数据信息的管理和维护，而 Executor 端的 Slave 需要将 Block 的更新等状态上报到 Master，同时接收 Master 的命令，例如新增或删除一个 RDD。

![image-20230208195829578](https://cos.gump.cloud/uPic/image-20230208195829578.png)

在对 RDD 持久化时，Spark 规定了 MEMORY_ONLY、MEMORY_AND_DISK 等 7 种不同的存储级别，而存储级别是以下 5 个变量的组合：

```java
class StorageLevel private(
    private var _useDisk: Boolean, //磁盘
    private var _useMemory: Boolean, //这里其实是指堆内内存
    private var _useOffHeap: Boolean, //堆外内存
    private var _deserialized: Boolean, //是否为非序列化
    private var _replication: Int = 1 //副本个数
)
```

Spark 中 7 种存储级别如下：

| 持久化级别                         | 含义                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| MEMORY_ONLY                        | 以非序列化的 Java 对象的方式持久化在 JVM 内存中。如果内存无法完全存储 RDD 所有的 partition，那么那些没有持久化的 partition 就会在下一次需要使用它们的时候，重新被计算 |
| MEMORY_AND_DISK                    | 同上，但是当某些 partition 无法存储在内存中时，会持久化到磁盘中。下次需要使用这些 partition 时，需要从磁盘上读取 |
| MEMORY_ONLY_SER                    | 同 MEMORY_ONLY，但是会使用 Java 序列化方式，将 Java 对象序列化后进行持久化。可以减少内存开销，但是需要进行反序列化，因此会加大 CPU 开销 |
| MEMORY_AND_DISK_SER                | 同 MEMORY_AND_DISK ，但是使用序列化方式持久化 Java 对象      |
| DISK_ONLY                          | 使用非序列化 Java 对象的方式持久化，完全存储到磁盘上         |
| MEMORY_ONLY_2MEMORY_AND_DISK_2等等 | 如果是尾部加了 2 的持久化级别，表示将持久化数据复用一份，保存到其他节点，从而在数据丢失时，不需要再次计算，只需要使用备份数据即可 |

通过对数据结构的分析，可以看出存储级别从三个维度定义了 RDD 的 Partition（同时也就是 Block ）的存储方式：

- 存储位置：磁盘／堆内内存／堆外内存。如 MEMORY_AND_DISK 是同时在磁盘和堆内内存上存储，实现了冗余备份。OFF_HEAP 则是只在堆外内存存储，目前选择堆外内存时不能同时存储到其他位置
- 存储形式：Block 缓存到存储内存后，是否为非序列化的形式。如 MEMORY_ONLY 是非序列化方式存储，OFF_HEAP 是序列化方式存储
- 副本数量：大于 1 时需要远程冗余备份到其他节点。如 DISK_ONLY_2 需要远程备份 1 个副本

### 15.3.2 淘汰与落盘

由于同一个 Executor 的所有的计算任务共享有限的存储内存空间，当有新的 Block 需要缓存但是剩余空间不足且无法动态占用时，就要对 LinkedHashMap 中的旧 Block 进行淘汰（Eviction），而被淘汰的 Block 如果其存储级别中同时包含存储到磁盘的要求，则要对其进行落盘（Drop），否则直接删除该 Block。

存储内存的淘汰规则为：

- 被淘汰的旧 Block 要与新 Block 的 MemoryMode 相同，即同属于堆外或堆内内存
- 新旧 Block 不能属于同一个 RDD，避免循环淘汰
- 旧 Block 所属 RDD 不能处于被读状态，避免引发一致性问题
- 遍历 LinkedHashMap 中 Block，按照最近最少使用（ LRU ）的顺序淘汰，直到满足新 Block 所需的空间。其中 LRU 是 LinkedHashMap 的特性

落盘的流程则比较简单，如果其存储级别符合 \_useDisk 为 true 的条件，再根据其 _deserialized 判断是否是非序列化的形式，若是则对其进行序列化，最后将数据存储到磁盘，在 Storage 模块中更新其信息。

## 15.4 执行内存管理

执行内存主要用来存储任务在执行 Shuffle 时占用的内存，Shuffle 是按照一定规则对 RDD 数据重新分区的过程，我们来看 Shuffle 的 Write 和 Read 两阶段对执行内存的使用：

1）Shuffle Write

若在 map 端选择快速排序方式，在内存中存储数据时主要占用堆内执行空间。

在内存中存储数据时可以占用堆外或堆内执行空间，取决于用户是否开启了堆外内存以及堆外执行内存是否足够。

2）Shuffle Read

在对 reduce 端的数据进行聚合时，要将数据交给 Aggregator 处理，在内存中存储数据时占用堆内执行空间。

