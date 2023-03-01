# 第5章 Yarn资源调度器

思考：

1）如何管理集群资源？

2）如何给任务合理分配资源？

![image-20230301193002085](https://cos.gump.cloud/uPic/image-20230301193002085.png)

Yarn 是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的{==操作系统平台==}，而 MapReduce 等运算程序则相当于运行于操作系统之上的{==应用程序==}。

## 5.1 Yarn基础架构

 YARN 主要由 ResourceManager、NodeManager、ApplicationMaster 和 Container 等组件构成。

![image-20230301193108155](https://cos.gump.cloud/uPic/image-20230301193108155.png)

<figure markdown>
  ![image-20230301193208806](https://cos.gump.cloud/uPic/image-20230301193208806.png)
  <figcaption>yarn工作流程</figcaption>
</figure>
（1）MR 程序提交到客户端所在的节点。

（2）YarnRunner 向 ResourceManager 申请一个 Application。

（3）RM 将该应用程序的资源路径返回给 YarnRunner。

（4）该程序将运行所需资源提交到 HDFS 上。

（5）程序资源提交完毕后，申请运行 mrAppMaster。

（6）RM 将用户的请求初始化成一个 Task。

（7）其中一个 NodeManager 领取到 Task 任务。

（8）该 NodeManager 创建容器 Container，并产生 MRAppmaster。

（9）Container 从 HDFS 上拷贝资源到本地。

（10）MRAppmaster 向 RM 申请运行 MapTask 资源。

（11）RM 将运行 MapTask 任务分配给另外两个 NodeManager，另两个 NodeManager 分别领取任务并创建容器。

（12）MR 向两个接收到任务的 NodeManager 发送程序启动脚本，这两个 NodeManager 分别启动 MapTask，MapTask 对数据分区排序。

（13）MrAppMaster 等待所有 MapTask 运行完毕后，向 RM 申请容器，运行 ReduceTask。

（14）ReduceTask 向 MapTask 获取相应分区的数据。

（15）程序运行完毕后，MR 会向 RM 申请注销自己。

## 5.3 作业提交全过程

<figure markdown>
  ![image-20230301193507236](https://cos.gump.cloud/uPic/image-20230301193507236.png)
  <figcaption>HDFS、MapReduce、Yarn的关系</figcaption>
</figure>

<figure markdown>
  ![image-20230301193553363](https://cos.gump.cloud/uPic/image-20230301193553363.png)
  <figcaption>Yarn提交任务流程</figcaption>
</figure>

<figure markdown>
  ![image-20230301193629074](https://cos.gump.cloud/uPic/image-20230301193629074.png)
  <figcaption>HDFS、MapReduce提交流程</figcaption>
</figure>

作业提交全过程详解

（1）作业提交

第 1 步：Client 调用 job.waitForCompletion 方法，向整个集群提交 MapReduce 作业。

第 2 步：Client 向 RM 申请一个作业 id。

第 3 步：RM 给 Client 返回该 job 资源的提交路径和作业 id。

第 4 步：Client 提交 jar 包、切片信息和配置文件到指定的资源提交路径。

第 5 步：Client 提交完资源后，向 RM 申请运行 MrAppMaster。

（2）作业初始化

第 6 步：当 RM 收到 Client 的请求后，将该 job 添加到容量调度器中。

第 7 步：某一个空闲的 NM 领取到该 Job。

第 8 步：该 NM 创建 Container，并产生 MRAppmaster。

第 9 步：下载 Client 提交的资源到本地。

（3）任务分配

第 10 步：MrAppMaster 向 RM 申请运行多个 MapTask 任务资源。

第 11 步：RM 将运行 MapTask 任务分配给另外两个 NodeManager，另两个 NodeManager 分别领取任务并创建容器。

（4）任务运行

第 12 步：MR 向两个接收到任务的 NodeManager 发送程序启动脚本，这两个 NodeManager 分别启动 MapTask，MapTask 对数据分区排序。

第 13 步：MrAppMaster 等待所有 MapTask 运行完毕后，向 RM 申请容器，运行 ReduceTask。

第 14 步：ReduceTask 向 MapTask 获取相应分区的数据。

第 15 步：程序运行完毕后，MR 会向 RM 申请注销自己。

（5）进度和状态更新

YARN 中的任务将其进度和状态(包括 counter )返回给应用管理器, 客户端每秒(通过 `mapreduce.client.progressmonitor.pollinterval` 设置)向应用管理器请求进度更新, 展示给用户。

（6）作业完成

除了向应用管理器请求作业进度外, 客户端每 5 秒都会通过调用 waitForCompletion() 来检查作业是否完成。时间间隔可以通过`mapreduce.client.completion.pollinterval` 来设置。作业完成之后, 应用管理器和 Container 会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。

## 5.4 Yarn调度器和调度算法

目前，Hadoop 作业调度器主要有三种：FIFO、容量（Capacity Scheduler）和公平（Fair Scheduler）。Apache Hadoop 3.1.3 默认的资源调度器是 {==Capacity Scheduler==}。

CDH 框架默认调度器是 Fair Scheduler。

```xml title="yarn-site.xml"
<property>
    <description>The class to use as the resource scheduler.</description>
    <name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

### 5.4.1 先进先出调度器（FIFO）

FIFO 调度器（First In First Out）：单队列，根据提交作业的先后顺序，先来先服务。

![image-20230301194155000](https://cos.gump.cloud/uPic/image-20230301194155000.png)

优点：简单易懂。

缺点：不支持多队列，生产环境很少使用。

### 5.4.2 容量调度器（Capacity Scheduler）

Capacity Scheduler 是 Yahoo 开发的多用户调度器。

![image-20230301194233287](https://cos.gump.cloud/uPic/image-20230301194233287.png)

![image-20230301194338487](https://cos.gump.cloud/uPic/image-20230301194338487.png)

### 5.4.3 公平调度器（Fair Scheduler）

Fair Scheduler 是 Facebook 开发的多用户调度器。

![image-20230301194444178](https://cos.gump.cloud/uPic/image-20230301194444178.png)

![image-20230301194645016](https://cos.gump.cloud/uPic/image-20230301194645016.png)

![image-20230301194655858](https://cos.gump.cloud/uPic/image-20230301194655858.png)

![image-20230301194902259](https://cos.gump.cloud/uPic/image-20230301194902259.png)

![image-20230301194912844](https://cos.gump.cloud/uPic/image-20230301194912844.png)

![image-20230301194921315](https://cos.gump.cloud/uPic/image-20230301194921315.png)