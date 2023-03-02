# 第3章 HDFS—故障排除

## 3.1 NameNode故障处理

![image-20230302104547458](https://cos.gump.cloud/uPic/image-20230302104547458.png)

**1）需求**

NameNode 进程挂了并且存储的数据也丢失了，如何恢复 NameNode

**2）故障模拟**

（1）kill -9 NameNode进程

```shell
kill -9 19886
```

（2）删除 NameNode 存储的数据（ /opt/module/hadoop-3.1.3/data/tmp/dfs/name ）

```shell
rm -rf /opt/module/hadoop-3.1.3/data/dfs/name/*
```

**3）问题解决**

（1）拷贝 SecondaryNameNode 中数据到原 NameNode 存储数据目录

```shell
scp -r atguigu@hadoop104:/opt/module/hadoop-3.1.3/data/dfs/namesecondary/* ./name/
```

（2）重新启动 NameNode

```shell
hdfs --daemon start namenode
```

（3）向集群上传一个文件

## 3.2 集群安全模式&磁盘修复

1）安全模式：文件系统只接受读数据请求，而不接受删除、修改等变更请求

2）进入安全模式场景

- NameNode 在加载镜像文件和编辑日志期间处于安全模式
- NameNode 在接收 DataNode 注册时，处于安全模式

![image-20230302104758514](https://cos.gump.cloud/uPic/image-20230302104758514.png)

3）退出安全模式条件

`dfs.namenode.safemode.threshold-pct`: 副本数达到最小要求的 block 占系统总 block 数的百分比，默认 0.999f。（只允许丢一个块）

4）基本语法

集群处于安全模式，不能执行重要操作（写操作）。集群启动完成后，自动退出安全模式。

```shell
（1）bin/hdfs dfsadmin -safemode get	（功能描述：查看安全模式状态）
（2）bin/hdfs dfsadmin -safemode enter （功能描述：进入安全模式状态）
（3）bin/hdfs dfsadmin -safemode leave	（功能描述：离开安全模式状态）
（4）bin/hdfs dfsadmin -safemode wait	（功能描述：等待安全模式状态）
```

5）案例 1：启动集群进入安全模式

（1）重新启动集群

```shell
myhadoop.sh stop
myhadoop.sh start
```

（2）集群启动后，立即来到集群上删除数据，提示集群处于安全模式

![image-20230302110910672](https://cos.gump.cloud/uPic/image-20230302110910672.png)

6）案例2：磁盘修复

​	需求：数据块损坏，进入安全模式，如何处理

（1）分别进入 hadoop102、hadoop103、hadoop104 的 /opt/module/hadoop-3.1.3/data/dfs/data/current/{==BP-1015489500-192.168.10.102-1611909480872==}/current/finalized/subdir0/subdir0 目录，统一删除某 2 个块信息

```shell
rm -rf blk_1073741847 blk_1073741847_1023.meta
rm -rf blk_1073741865 blk_1073741865_1042.meta
```

说明：hadoop103 / hadoop104 重复执行以上命令

（2）重新启动集群

```shell
myhadoop.sh stop
myhadoop.sh start
```

（3）观察 http://hadoop102:9870/dfshealth.html#tab-overview

![image-20230302144226667](https://cos.gump.cloud/uPic/image-20230302144226667.png)

!!! info "说明"

    安全模式已经打开，块的数量没有达到要求。

（4）离开安全模式

```shell
hdfs dfsadmin -safemode get
Safe mode is ON

hdfs dfsadmin -safemode leave
Safe mode is OFF
```

（5）观察 http://hadoop102:9870/dfshealth.html#tab-overview

![image-20230302144333339](https://cos.gump.cloud/uPic/image-20230302144333339.png)

（6）将元数据删除

![image-20230302144345317](https://cos.gump.cloud/uPic/image-20230302144345317.png)

![image-20230302144349281](https://cos.gump.cloud/uPic/image-20230302144349281.png)

（7）观察 http://hadoop102:9870/dfshealth.html#tab-overview，集群已经正常

7）案例3：

需求：模拟等待安全模式。

（1）查看当前模式

```shell
hdfs dfsadmin -safemode get
Safe mode is OFF
```

（2）先进入安全模式

```shell
bin/hdfs dfsadmin -safemode enter
```

（3）创建并执行下面的脚本

在 /opt/module/hadoop-3.1.3 路径上，编辑一个脚本 safemode.sh。

```shell title="safemode.sh"
#!/bin/bash
hdfs dfsadmin -safemode wait
hdfs dfs -put /opt/module/hadoop-3.1.3/README.txt /
```

（4）再打开一个窗口，执行

```shell
bin/hdfs dfsadmin -safemode leave
```

（5）再观察上一个窗口

```shell
Safe mode is OFF
```

（6）HDFS 集群上已经有上传的数据了

![image-20230302144606043](https://cos.gump.cloud/uPic/image-20230302144606043.png)
