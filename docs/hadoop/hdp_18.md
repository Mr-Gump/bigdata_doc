# 第1章 HDFS—多目录

## 1.1 DataNode多目录配置

1）DataNode 可以配置成多个目录，{==每个目录存储的数据不一样==}（数据不是副本）

![image-20230302102918137](https://cos.gump.cloud/uPic/image-20230302102918137.png)

2）具体配置如下

在 hdfs-site.xml 文件中添加如下内容。

```xml title="hdfs-site.xml"
<property>
     <name>dfs.datanode.data.dir</name>
     <value>file://${hadoop.tmp.dir}/dfs/data1,file://${hadoop.tmp.dir}/dfs/data2</value>
</property>
```

3）查看结果

```shell
ll

总用量 12
drwx------. 3 atguigu atguigu 4096 4月   4 14:22 data1
drwx------. 3 atguigu atguigu 4096 4月   4 14:22 data2
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name1
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name2
```

4）向集群上传一个文件，再次观察两个文件夹里面的内容发现不一致（一个有数一个没有）

```shell
hadoop fs -put wcinput/word.txt /
```

## 1.2 集群数据均衡之磁盘间数据均衡

生产环境，由于硬盘空间不足，往往需要增加一块硬盘。刚加载的硬盘没有数据时，可以执行磁盘数据均衡命令。（{==Hadoop3.x 新特性==}）。

![image-20230302103052073](https://cos.gump.cloud/uPic/image-20230302103052073.png)

（1）生成均衡计划（我们只有一块磁盘，不会生成计划）

```shell
hdfs diskbalancer -plan hadoop103
```

（2）执行均衡计划

```shell
hdfs diskbalancer -execute hadoop103.plan.json
```

（3）查看当前均衡任务的执行情况

```shell
hdfs diskbalancer -query hadoop103
```

（4）取消均衡任务

```shell
hdfs diskbalancer -cancel hadoop103.plan.json
```

