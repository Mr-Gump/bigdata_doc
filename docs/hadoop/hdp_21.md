# 第4章Hadoop企业优化

## 4.1 MapReduce优化方法

MapReduce 优化方法主要从六个方面考虑：数据输入、Map 阶段、Reduce 阶段、IO 传输、数据倾斜问题和常用的调优参数。

### 4.1.1 数据输入

![image-20230302144650966](https://cos.gump.cloud/uPic/image-20230302144650966.png)

### 4.1.2 Map阶段

![image-20230302144709298](https://cos.gump.cloud/uPic/image-20230302144709298.png)

### 4.1.3 Reduce阶段

![image-20230302144724719](https://cos.gump.cloud/uPic/image-20230302144724719.png)

![image-20230302144736027](https://cos.gump.cloud/uPic/image-20230302144736027.png)

### 4.1.4 I/O传输

![image-20230302144748638](https://cos.gump.cloud/uPic/image-20230302144748638.png)

![image-20230302144758909](https://cos.gump.cloud/uPic/image-20230302144758909.png)

![image-20230302144808391](https://cos.gump.cloud/uPic/image-20230302144808391.png)

## 4.2 常用的调优参数

**1）资源相关参数**

（1）以下参数是在用户自己的 MR 应用程序中配置就可以生效（mapred-default.xml）

| 配置参数                                      | 参数说明                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| mapreduce.map.memory.mb                       | 一个MapTask可使用的资源上限（单位:MB），默认为1024。如果MapTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.reduce.memory.mb                    | 一个ReduceTask可使用的资源上限（单位:MB），默认为1024。如果ReduceTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.map.cpu.vcores                      | 每个MapTask可使用的最多cpu core数目，默认值: 1               |
| mapreduce.reduce.cpu.vcores                   | 每个ReduceTask可使用的最多cpu core数目，默认值: 1            |
| mapreduce.reduce.shuffle.parallelcopies       | 每个Reduce去Map中取数据的并行数。默认值是5                   |
| mapreduce.reduce.shuffle.merge.percent        | Buffer中的数据达到多少比例开始写入磁盘。默认值0.66           |
| mapreduce.reduce.shuffle.input.buffer.percent | Buffer大小占Reduce可用内存的比例。默认值0.7                  |
| mapreduce.reduce.input.buffer.percent         | 指定多少比例的内存用来存放Buffer中的数据，默认值是0.0        |

（2）应该在 YARN 启动之前就配置在服务器的配置文件中才能生效（yarn-default.xml）

| 配置参数                                 | 参数说明                                        |
| ---------------------------------------- | ----------------------------------------------- |
| yarn.scheduler.minimum-allocation-mb     | 给应用程序Container分配的最小内存，默认值：1024 |
| yarn.scheduler.maximum-allocation-mb     | 给应用程序Container分配的最大内存，默认值：8192 |
| yarn.scheduler.minimum-allocation-vcores | 每个Container申请的最小CPU核数，默认值：1       |
| yarn.scheduler.maximum-allocation-vcores | 每个Container申请的最大CPU核数，默认值：32      |
| yarn.nodemanager.resource.memory-mb      | 给Containers分配的最大物理内存，默认值：8192    |

（3）Shuffle 性能优化的关键参数，应在 YARN 启动之前就配置好（mapred-default.xml）

| 配置参数                         | 参数说明                          |
| -------------------------------- | --------------------------------- |
| mapreduce.task.io.sort.mb        | Shuffle的环形缓冲区大小，默认100m |
| mapreduce.map.sort.spill.percent | 环形缓冲区溢出的阈值，默认80%     |

**2）容错相关参数（MapReduce 性能优化）**

| 配置参数                     | 参数说明                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| mapreduce.map.maxattempts    | 每个Map Task最大重试次数，一旦重试次数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.reduce.maxattempts | 每个Reduce Task最大重试次数，一旦重试次数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.task.timeout       | Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个Task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该Task处于Block状态，可能是卡住了，也许永远会卡住，为了防止因为用户程序永远Block住不退出，则强制设置了一个该超时时间（单位毫秒），默认是600000（10分钟）。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是：“AttemptID:attempt_14267829456721_123456_m_000224_0 Timed out after 300 secsContainer killed by the ApplicationMaster.”。 |

## 4.4 Hadoop小文件优化方法

### 4.4.1 Hadoop小文件弊端

HDFS 上每个文件都要在 NameNode 上创建对应的元数据，这个元数据的大小约为 150byte，这样当小文件比较多的时候，就会产生很多的元数据文件，{==一方面会大量占用 NameNode 的内存空间，另一方面就是元数据文件过多，使得寻址索引速度变慢==}。

小文件过多，在进行 MR 计算时，会生成过多切片，需要启动过多的 MapTask。每个 MapTask 处理的数据量小，{==导致 MapTask 的处理时间比启动时间还小，白白消耗资源==}。

### 4.4.2Hadoop小文件解决方案

1）小文件优化的方向：

（1）在数据采集的时候，就将小文件或小批数据合成大文件再上传 HDFS。

（2）在业务处理之前，在 HDFS 上使用 MapReduce 程序对小文件进行合并。

（3）在 MapReduce 处理时，可采用 CombineTextInputFormat 提高效率。

（4）开启 uber 模式，实现 jvm 重用

2）Hadoop Archive

是一个高效的将小文件放入 HDFS 块中的文件存档工具，能够将多个小文件打包成一个 HAR 文件，从而达到减少 NameNode 的内存使用

3）CombineTextInputFormat

CombineTextInputFormat 用于将多个小文件在切片过程中生成一个单独的切片或者少量的切片。 

4）开启 uber 模式，实现 JVM 重用。

默认情况下，每个 Task 任务都需要启动一个 JVM 来运行，如果 Task 任务计算的数据量很小，我们可以让同一个 Job 的多个 Task 运行在一个 JVM 中，不必为每个 Task 都开启一个 JVM. 

开启 uber 模式，在 mapred-site.xml 中添加如下配置

```xml title="mapred-site.xml"
<!--  开启uber模式 -->
<property>
	<name>mapreduce.job.ubertask.enable</name>
	<value>true</value>
</property>

<!-- uber模式中最大的mapTask数量，可向下修改  --> 
<property>
	<name>mapreduce.job.ubertask.maxmaps</name>
	<value>9</value>
</property>
<!-- uber模式中最大的reduce数量，可向下修改 -->
<property>
	<name>mapreduce.job.ubertask.maxreduces</name>
	<value>1</value>
</property>
<!-- uber模式中最大的输入数据量，默认使用dfs.blocksize 的值，可向下修改 -->
<property>
	<name>mapreduce.job.ubertask.maxbytes</name>
	<value></value>
</property>
```

