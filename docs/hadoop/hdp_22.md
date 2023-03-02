# 第5章 Hadoop扩展（新特性）

## 5.1 集群间数据拷贝

**1）scp 实现两个远程主机之间的文件复制**

```shell
	scp -r hello.txt root@hadoop103:/user/atguigu/hello.txt		// 推 push
	scp -r root@hadoop103:/user/atguigu/hello.txt  hello.txt		// 拉 pull
	scp -r root@hadoop103:/user/atguigu/hello.txt root@hadoop104:/user/atguigu   //是通过本地主机中转实现两个远程主机的文件复制；如果在两个远程主机之间ssh没有配置的情况下可以使用该方式。
```

**2）采用 distcp 命令实现两个 Hadoop 集群之间的递归数据复制**

```shell
bin/hadoop distcp hdfs://hadoop102:8020/user/atguigu/hello.txt hdfs://hadoop105:8020/user/atguigu/hello.txt
```

## 5.2 小文件存档

![image-20230302145458575](https://cos.gump.cloud/uPic/image-20230302145458575.png)

**1）案例实操**

（1）需要启动YARN进程

```shell
start-yarn.sh
```

（2）归档文件

把 /user/atguigu/input 目录里面的所有文件归档成一个叫 input.har 的归档文件，并把归档后文件存储到 /user/atguigu/output 路径下。

```shell
hadoop archive -archiveName input.har -p  /user/atguigu/input   /user/atguigu/output
```

（3）查看归档

```shell
hadoop fs -ls har:///user/atguigu/output/input.har
```

（4）解归档文件

```shell
hadoop fs -cp har:///user/atguigu/output/input.har/*    /user/atguigu
```

## 5.3 回收站

开启回收站功能，可以将删除的文件在不超时的情况下，恢复原数据，起到防止误删除、备份等作用。

**1）回收站参数设置及工作机制**

![image-20230302145737165](https://cos.gump.cloud/uPic/image-20230302145737165.png)

**2）启用回收站**

修改 core-site.xml，配置垃圾回收时间为 1 分钟。

```xml title="core-site.xml"
<property>
    <name>fs.trash.interval</name>
        <value>1</value>
    </property>
    <property>
    <name>fs.trash.checkpoint.interval</name>
        <value>1</value>
</property>
```

**3）查看回收站**

回收站目录在 hdfs 集群中的路径：/user/atguigu/.Trash/….

**4）通过程序删除的文件不会经过回收站，需要调用 moveToTrash() 才进入回收站**

```java
//因为本地的客户端拿不到集群的配置信息 所以需要自己手动设置一下回收站
conf.set(“fs.trash.interval”,”1”);
conf.set(“fs.trash.checkpoint.interval”,”1”);
//创建一个回收站对象
Trash trash = New Trash(conf);
trash.moveToTrash(path);
```

**5）通过网页上直接删除的文件也不会走回收站。**

**6）只有在命令行利用 `hadoop fs -rm` 命令删除的文件才会走回收站。**

```shell
hadoop fs -rm -r /user/atguigu/input

2020-07-14 16:13:42,643 INFO fs.TrashPolicyDefault: Moved: 'hdfs://hadoop102:8020/user/atguigu/input' to trash at: hdfs://hadoop102:8020/user/atguigu/.Trash/Current/user/atguigu/input
```

**7）恢复回收站数据**

```shell
hadoop fs -mv
/user/atguigu/.Trash/Current/user/atguigu/input    /user/atguigu/input
```

