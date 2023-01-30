# 第2章 Spark运行模式

部署 Spark 集群大体上分为两种模式：{++单机模式++}与{++集群模式++}



大多数分布式框架都支持单机模式，方便开发者调试框架的运行环境。但是在生产环境中，并不会使用单机模式。因此，后续直接按照集群模式部署 Spark 集群。



下面详细列举了 Spark 目前支持的部署模式:

+ Local 模式：在本地部署单个Spark服务
+ Standalone 模式：Spark 自带的任务调度模式。（国内常用）
+ YARN 模式：Spark 使用 Hadoop 的 YARN 组件进行资源与任务调度。（国内最常用）
+ Mesos 模式：Spark 使用 Mesos 平台进行资源与任务的调度。（国内很少用）



## 2.1 Spark安装地址

| 参考资料 | 链接                                                       |
| :------: | ---------------------------------------------------------- |
| 官网地址 | [:material-link:](http://spark.apache.org/)                |
| 文档地址 | [:material-link:](https://spark.apache.org/docs/3.3.1/)    |
| 下载地址 | [:material-link:](https://spark.apache.org/downloads.html) |

## 2.2 Local模式

Local 模式就是运行在一台计算机上的模式，通常就是用于在本机上练手和测试。

### 2.2.1 安装使用

1）上传并解压 Spark 安装包

```bash
tar -zxvf spark-3.1.3-bin-hadoop3.2.tgz -C /opt/module/
mv spark-3.1.3-bin-hadoop3.2 spark-local
```

2）官方求 PI 案例

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[2] \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

- --class：表示要执行程序的主类；

- --master local[2]
- - local: 没有指定线程数，则所有计算都运行在一个线程当中，没有任何并行计算
  - local[K]:指定使用 K 个 Core 来运行计算，比如 local[2] 就是运行 2 个 Core 来执行
  - local[*]：默认模式。自动帮你按照 CPU 最多核来设置线程数。比如 CPU 有 8 核，Spark 帮你自动设置 8 个线程计算。
- spark-examples_2.12-3.1.3.jar：要运行的程序；
- 10：要运行程序的输入参数（计算圆周率 π 的次数，计算次数越多，准确率越高）



3）结果展示

![image-20230130203539884](https://cos.gump.cloud/uPic/image-20230130203539884.png)



### 2.2.2 查看任务运行详情

1）再次运行求 PI 任务，增加迭代次数

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[2] \
./examples/jars/spark-examples_2.12-3.1.3.jar \
1000
```

2）在任务运行还没有完成时，可登录 hadoop102:4040 查看程序运行结果

![image-20230130203700543](https://cos.gump.cloud/uPic/image-20230130203700543.png)



## 2.3 Standalone模式

Standalone 模式是 Spark 自带的资源调度引擎，构建一个由 {++Master + Worker++} 构成的 Spark 集群，Spark 运行在集群中。
这个要和 Hadoop 中的 Standalone 区别开来。这里的 Standalone 是指只用 Spark 来搭建一个集群，不需要借助 Hadoop 的 Yarn 和 Mesos 等其他框架。



### 2.3.1集群角色

#### 2.3.1.1 Master和Worker——集群资源管理

#### ![image-20230130203858648](https://cos.gump.cloud/uPic/image-20230130203858648.png) 

Master 和 Worker 是 Spark 的{++守护进程++}、{++集群资源管理者++}，即 Spark 在特定模式（Standalone）下正常运行必须要有的{==后台常驻进程==}。

#### 2.3.1.2 Driver和Executor——任务的管理者

![image-20230130204051903](https://cos.gump.cloud/uPic/image-20230130204051903.png)

Driver 和 Executor 是{++临时程序++}，当有具体任务提交到 Spark 集群才会开启的程序。

### 2.3.2 安装使用

1）集群规划

|       | hadoop102            | hadoop103 | hadoop104 |
| ----- | -------------------- | --------- | --------- |
| Spark | {==Master==}、Worker | Worker    | Worker    |

2）再解压一份 Spark 安装包，并修改解压后的文件夹名称为 spark-standalone

```bash
tar -zxvf spark-3.1.3-bin-hadoop3.2.tgz -C /opt/module/
mv spark-3.1.3-bin-hadoop3.2 spark-standalone
```

3）进入 Spark 的配置目录 /opt/module/spark-standalone/conf

```bash
cd conf
```

4）修改 workers 文件，添加 work 节点：

```bash
mv workers.template workers
vim workers
```

``` conf title="workers"
hadoop102
hadoop103
hadoop104
```

5）修改 spark-env.sh 文件，添加 master 节点

```bash
mv spark-env.sh.template spark-env.sh
vim spark-env.sh
```

```conf title="spark-env.sh"
SPARK_MASTER_HOST=hadoop102
SPARK_MASTER_PORT=7077
```

6）分发 spark-standalone 包

```bash
xsync spark-standalone/
```

7）启动 spark 集群

```bash
sbin/start-all.sh
```

查看三台服务器运行进程（ xcall.sh 是以前数仓项目里面讲的脚本）

<div class="termy">
```console
$ xcall.sh jps
================atguigu@hadoop102================
3238 Worker
3163 Master
================atguigu@hadoop103================
2908 Worker
================atguigu@hadoop104================
2978 Worker
```
</div>

!!! warning "注意"
    如果遇到 `JAVA_HOME not set` 异常，可以在 sbin 目录下的 spark-config.sh 文件中加入如下配置。
    ``` conf title="spark-config.sh"
    export JAVA_HOME=XXXX
    ```
    
8）网页查看：hadoop102:8080（master web 的端口，相当于 Yarn 的8088端口）

目前还看不到任何任务的执行信息。

9）官方求 PI 案例
```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077 \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

参数：--master spark://hadoop102:7077 指定要连接的集群的 master。

10）结果查看

页面查看 http://hadoop102:8080/ ，发现执行本次任务，默认采用三台服务器节点的总核数 24 核，每个节点内存 1024 M。
- 8080：master 的 webUI。
- 4040：application 的 webUI的端口号。

![image-20230130205119430](https://cos.gump.cloud/uPic/image-20230130205119430.png)

### 2.3.3 参数说明

1）配置 Executor 可用内存为 2G，使用 CPU 核数为 2 个

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077 \
--executor-memory 2G \
--total-executor-cores 2 \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

2）页面查看 http://hadoop102:8080/

![image-20230130205217318](https://cos.gump.cloud/uPic/image-20230130205217318.png)

3）基本语法

```bash
bin/spark-submit \
--class <main-class>
--master <master-url> \
... # other options
<application-jar> \
[application-arguments]
```

4）参数说明

| 参数                     |                             解释                             | 可选值举例                                       |
| ------------------------ | :----------------------------------------------------------: | ------------------------------------------------ |
| --class                  |                  Spark 程序中包含主函数的类                  |                                                  |
| --master                 |                     Spark 程序运行的模式                     | 本地模式：local[*]、spark://hadoop102:7077、Yarn |
| --executor-memory 1G     |               指定每个 executor 可用内存为 1G                | 符合集群内存配置即可，具体情况具体分析。         |
| --total-executor-cores 2 |           指定所有 executor 使用的 cpu 核数为 2 个           |                                                  |
| application-jar          | 打包好的应用 jar，包含依赖。这个 URL 在集群中全局可见。 比如 hdfs:// 共享存储系统，如果是 file:// path，那么所有的节点的 path 都包含同样的 jar |                                                  |
| application-arguments    |                    传给 main() 方法的参数                    |                                                  |

### 2.3.4 配置历史服务

由于 spark 任务停止掉后，hadoop102:4040 页面就看不到历史任务的运行情况，所以开发时都配置历史服务器记录任务运行情况。

1）修改 spark-default.conf.template 名称

```bash
 mv spark-defaults.conf.template spark-defaults.conf
```

2）修改 spark-default.conf 文件，配置日志存储路径（写）

```bash
vim spark-defaults.conf
```

``` conf title="spark-defaults.conf"
spark.eventLog.enabled          true
spark.eventLog.dir              hdfs://hadoop102:8020/directory
```

!!! warning "注意"

    需要启动 Hadoop 集群，HDFS 上的目录需要提前存在。


    ``` bash
    sbin/start-dfs.sh
    hadoop fs -mkdir /directory
    ```

3）修改spark-env.sh文件，添加如下配置：
```bash
vim spark-env.sh
```

```bash title="spark-env.sh"
export SPARK_HISTORY_OPTS="
-Dspark.history.ui.port=18080 
-Dspark.history.fs.logDirectory=hdfs://hadoop102:8020/directory 
-Dspark.history.retainedApplications=30"
```

- 参数 1 含义：WEBUI 访问的端口号为 18080
- 参数 2 含义：指定历史服务器日志存储路径（读）
- 参数 3 含义：指定保存 Application 历史记录的个数，如果超过这个值，旧的应用程序信息将被删除，这个是内存中的应用数，而不是页面上显示的应用数。

4）分发配置文件
```bash
xsync spark-defaults.conf spark-env.sh
```
5）启动历史服务
```bash
sbin/start-history-server.sh
```

6）再次执行任务
```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077 \
--executor-memory 1G \
--total-executor-cores 2 \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

7）查看 Spark 历史服务地址：hadoop102:18080

![image-20230130210504237](https://cos.gump.cloud/uPic/image-20230130210504237.png)

### 2.3.5 配置高可用（HA）

1）高可用原理

![image-20230130210548552](https://cos.gump.cloud/uPic/image-20230130210548552.png)

2）配置高可用

（0）停止集群

```bash
sbin/stop-all.sh
```

（1）Zookeeper正常安装并启动（基于以前讲的数仓项目脚本）

```bash
 zk.sh start
```

（2）修改spark-env.sh文件添加如下配置：

```bash title="spark-env.sh"
#注释掉如下内容：
#SPARK_MASTER_HOST=hadoop102
#SPARK_MASTER_PORT=7077

#添加上如下内容。配置由Zookeeper管理Master，在Zookeeper节点中自动创建/spark目录，用于管理：
export SPARK_DAEMON_JAVA_OPTS="
-Dspark.deploy.recoveryMode=ZOOKEEPER 
-Dspark.deploy.zookeeper.url=hadoop102,hadoop103,hadoop104 
-Dspark.deploy.zookeeper.dir=/spark"

#添加如下代码
#Zookeeper3.5的AdminServer默认端口是8080，和Spark的WebUI冲突
export SPARK_MASTER_WEBUI_PORT=8989
```

（3）分发配置文件

```bash
xsync spark-env.sh
```

（4）在 hadoop102 上启动全部节点

```bash
sbin/start-all.sh
```

（5）在 hadoop103 上单独启动 master 节点

```bash
sbin/start-master.sh
```

3）高可用测试

![image-20230130210824361](https://cos.gump.cloud/uPic/image-20230130210824361.png)

![image-20230130210833306](https://cos.gump.cloud/uPic/image-20230130210833306.png)

​	（1）查看 hadoop102 的 master 进程

<div class="termy">
```console
$ jps
5506 Worker
5394 Master
5731 SparkSubmit
4869 QuorumPeerMain
5991 Jps
5831 CoarseGrainedExecutorBackend
```
</div>

（2）Kill 掉 hadoop102 的 master 进程，页面中观察 http://hadoop103:8080/ 的状态是否切换为 active。

![image-20230130211017491](https://cos.gump.cloud/uPic/image-20230130211017491.png)

```bash
kill -9 5394
```

（3）再启动 hadoop102 的 master 进程

```bash
sbin/start-master.sh
```

### 2.3.6 运行流程

Spark 有 standalone-client 和 standalone-cluster 两种模式，主要区别在于：Driver 程序的运行节点。

1）客户端模式

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077,hadoop103:7077 \
--executor-memory 2G \
--total-executor-cores 2 \
--deploy-mode client \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

--deploy-mode client，表示 Driver 程序运行在本地客户端，{==默认模式==}。

![image-20230130211233420](https://cos.gump.cloud/uPic/image-20230130211233420.png)

2）集群模式

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop102:7077,hadoop103:7077 \
--executor-memory 2G \
--total-executor-cores 2 \
--deploy-mode cluster \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

--deploy-mode cluster，表示 Driver 程序运行在集群。

![image-20230130211347259](https://cos.gump.cloud/uPic/image-20230130211347259.png)

​	（1）查看 http://hadoop102:8989/ 页面，点击 Completed Drivers 里面的 Worker。

![image-20230130211411340](https://cos.gump.cloud/uPic/image-20230130211411340.png)

（2）跳转到 Spark Worker 页面，点击 Finished Drivers 中 Logs 下面的 stdout。

![image-20230130211431892](https://cos.gump.cloud/uPic/image-20230130211431892.png)

（3）最终打印结果如下。

![image-20230130211440612](https://cos.gump.cloud/uPic/image-20230130211440612.png)

!!! info "注意"

    在测试 Standalone 模式，cluster 运行流程的时候，阿里云用户访问不到 Worker，因为 Worker 是从 Master 内部跳转的，这是正常的，实际工作中我们不可能通过客户端访问的，这些端口对外都会禁用，需要的时候会通过授权到 Master 访问 Worker

## 2.4 Yarn模式（重点）
Spark 客户端直接连接 Yarn，不需要额外构建 Spark 集群。

### 2.4.1 安装使用
0）停止 Standalone 模式下的 spark 集群
```bash
sbin/stop-all.sh
zk.sh stop
sbin/stop-master.sh
```

1）为了防止和 Standalone 模式冲突，再单独解压一份 spark
```bash
tar -zxvf spark-3.1.3-bin-hadoop3.2.tgz -C /opt/module/
```

2）进入到 /opt/module 目录，修改 spark-3.1.3-bin-hadoop3.2 名称为 spark-yarn
```bash
mv spark-3.1.3-bin-hadoop3.2/ spark-yarn
```

3）修改 hadoop 配置文件 /opt/module/hadoop/etc/hadoop/yarn-site.xml，添加如下内容

{==因为测试环境虚拟机内存较少，防止执行过程进行被意外杀死，做如下配置==}

```xml title="yarn-site.xml"
<!--是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
<property>
     <name>yarn.nodemanager.pmem-check-enabled</name>
     <value>false</value>
</property>

<!--是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
<property>
     <name>yarn.nodemanager.vmem-check-enabled</name>
     <value>false</value>
</property>
```

4）分发配置文件
```bash
xsync /opt/module/hadoop/etc/hadoop/yarn-site.xml
```

5）修改 /opt/module/spark-yarn/conf/spark-env.sh，添加 YARN_CONF_DIR 配置，保证后续运行任务的路径都变成集群路径
```bash
mv spark-env.sh.template spark-env.sh
```

```bash title="spark-env.sh"
YARN_CONF_DIR=/opt/module/hadoop/etc/hadoop
```

6）启动 HDFS 以及 YARN 集群
```bash
sbin/start-dfs.sh
sbin/start-yarn.sh
```

7）执行一个程序
```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

参数：--master yarn，表示 Yarn 方式运行；--deploy-mode 表示客户端方式运行程序

8）查看 hadoop103:8088 页面，点击 History，查看历史页面

![image-20230130212056535](https://cos.gump.cloud/uPic/image-20230130212056535.png)

!!! question "思考"
    目前是 Hadoop 的作业运行日志展示，如果想获取 Spark 的作业运行日志，怎么办？ 

### 2.4.2 配置历史服务

由于是重新解压的 Spark 压缩文件，所以需要针对 Yarn 模式，再次配置一下历史服务器。

1）修改 spark-default.conf.template 名称
```bash
mv spark-defaults.conf.template spark-defaults.conf
```

2）修改 spark-defaults.conf 文件，配置日志存储路径（写）
```conf title="spark-defaults.conf"
spark.eventLog.enabled          true
spark.eventLog.dir               hdfs://hadoop102:8020/directory
```

3）修改 spark-env.sh 文件，添加如下配置：
```bash title="spark-env.sh"
export SPARK_HISTORY_OPTS="
-Dspark.history.ui.port=18080 
-Dspark.history.fs.logDirectory=hdfs://hadoop102:8020/directory 
-Dspark.history.retainedApplications=30"
```

- 参数 1 含义：WEBUI 访问的端口号为 18080
- 参数 2 含义：指定历史服务器日志存储路径（读）
- 参数 3 含义：指定保存 Application 历史记录的个数，如果超过这个值，旧的应用程序信息将被删除，这个是内存中的应用数，而不是页面上显示的应用数。

### 2.4.3 配置查看历史日志
为了能从 Yarn 上关联到 spark 历史服务器，需要配置 spark 历史服务器关联路径。
目的：点击 yarn（8088）上 spark 任务的 history 按钮，进入的是 spark 历史服务器（18080），而不再是 yarn 历史服务器（19888）。

1）修改配置文件 /opt/module/spark-yarn/conf/spark-defaults.conf

添加如下内容：
```conf title="spark-defaults.conf"
spark.yarn.historyServer.address=hadoop102:18080
spark.history.ui.port=18080
```

2）重启 Spark 历史服务

```bash
sbin/stop-history-server.sh 
sbin/start-history-server.sh 
```

3）提交任务到 Yarn 执行
```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

4）Web 页面查看日志：http://hadoop103:8088/cluster

![image-20230130212625940](https://cos.gump.cloud/uPic/image-20230130212625940.png)

点击 `history` 跳转到 http://hadoop102:18080/

![image-20230130212629824](https://cos.gump.cloud/uPic/image-20230130212629824.png)

### 2.4.4 运行流程

Spark 有 yarn-client 和 yarn-cluster 两种模式，主要区别在于：Driver 程序的运行节点。

- yarn-client：Driver 程序运行在客户端，适用于交互、调试，希望立即看到 app 的输出。
- yarn-cluster：Driver 程序运行在由 ResourceManager 启动的 APPMaster，适用于生产环境。

1）客户端模式（默认）

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode client \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

![image-20230130212753228](https://cos.gump.cloud/uPic/image-20230130212753228.png)

2）集群模式

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
./examples/jars/spark-examples_2.12-3.1.3.jar \
10
```

（1）查看 http://hadoop103:8088/cluster 页面，点击 History 按钮，跳转到历史详情页面

![image-20230130212857311](https://cos.gump.cloud/uPic/image-20230130212857311.png)

（2）http://hadoop102:18080 点击 Executors -> 点击 driver 中的 stdout

![image-20230130212919416](https://cos.gump.cloud/uPic/image-20230130212919416.png)

![image-20230130212923516](https://cos.gump.cloud/uPic/image-20230130212923516.png)

![image-20230130212930083](https://cos.gump.cloud/uPic/image-20230130212930083.png)

!!! warning "注意"
    如果在 Yarn 日志端无法查看到具体的日志，则在 yarn-site.xml 中添加如下配置并启动 Yarn 历史服务器
    

![image-20230130213019903](https://cos.gump.cloud/uPic/image-20230130213019903.png)

```xml title="yarn-site.xml"
<property>
    <name>yarn.log.server.url</name>
    <value>http://hadoop102:19888/jobhistory/logs</value>
</property>
```

!!! info "注意"

    hadoop 历史服务器也要启动 mr-jobhistory-daemon.sh start historyserver

![image-20230130213209264](https://cos.gump.cloud/uPic/image-20230130213209264.png)

## 2.5 Mesos模式（了解）

Spark 客户端直接连接 Mesos；不需要额外构建 Spark 集群。国内应用比较少，更多的是运用 Yarn 调度。

## 2.6 几种模式对比

| 模式       | Spark安装机器数 | 需启动的进程   | 所属者 |
| ---------- | --------------- | -------------- | :----: |
| Local      | 1               | 无             | Spark  |
| Standalone | 3               | Master及Worker | Spark  |
| Yarn       | 1               | Yarn及HDFS     | Hadoop |

## 2.7 端口号总结

1. Spark 查看当前 Spark-shell 运行任务情况端口号：4040
2. Spark Master 内部通信服务端口号：7077	（类比于 Yarn 的 8032 端口（ RM 和 NM 的内部通信））
3. Spark Standalone 模式 Master Web 端口号：8080（类比于 Hadoop Yarn 任务运行情况查看端口号：8088）
4. Spark 历史服务器端口号：18080		（类比于 Hadoop 历史服务器端口号：19888）
