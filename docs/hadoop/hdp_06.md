# 第2章 HDFS的Shell操作（开发重点）

## 2.1 基本语法

`hadoop fs 具体命令`  或  `hdfs dfs 具体命令`

两个是完全相同的。

## 2.2 命令大全

<div class="termy">
```console
$ bin/hadoop fs

[-appendToFile <localsrc> ... <dst>]
        [-cat [-ignoreCrc] <src> ...]
        [-chgrp [-R] GROUP PATH...]
        [-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
        [-chown [-R] [OWNER][:[GROUP]] PATH...]
        [-copyFromLocal [-f] [-p] <localsrc> ... <dst>]
        [-copyToLocal [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-count [-q] <path> ...]
        [-cp [-f] [-p] <src> ... <dst>]
        [-df [-h] [<path> ...]]
        [-du [-s] [-h] <path> ...]
        [-get [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-getmerge [-nl] <src> <localdst>]
        [-help [cmd ...]]
        [-ls [-d] [-h] [-R] [<path> ...]]
        [-mkdir [-p] <path> ...]
        [-moveFromLocal <localsrc> ... <dst>]
        [-moveToLocal <src> <localdst>]
        [-mv <src> ... <dst>]
        [-put [-f] [-p] <localsrc> ... <dst>]
        [-rm [-f] [-r|-R] [-skipTrash] <src> ...]
        [-rmdir [--ignore-fail-on-non-empty] <dir> ...]
<acl_spec> <path>]]
        [-setrep [-R] [-w] <rep> <path> ...]
        [-stat [format] <path> ...]
        [-tail [-f] <file>]
        [-test -[defsz] <path>]
        [-text [-ignoreCrc] <src> ...]
```
</div>

## 2.3 常用命令实操


### 2.3.1 准备工作

1）启动 Hadoop 集群（方便后续的测试）

```shell
sbin/start-dfs.sh
sbin/start-yarn.sh
```

2）-help：输出这个命令参数

```shell
hadoop fs -help rm
```

3）创建 /sanguo 文件夹

```shell
hadoop fs -mkdir /sanguo
```

### 2.3.2 上传

1）-moveFromLocal：从本地剪切粘贴到 HDFS

```shell
hadoop fs  -moveFromLocal  ./shuguo.txt  /sanguo
```

2）-copyFromLocal：从本地文件系统中拷贝文件到 HDFS 路径去

```shell
hadoop fs -copyFromLocal weiguo.txt /sanguo
```

3）-put：等同于 copyFromLocal，生产环境更习惯用 put

```shell
hadoop fs -put ./wuguo.txt /sanguo
```

4）-appendToFile：追加一个文件到已经存在的文件末尾

```shell
hadoop fs -appendToFile liubei.txt /sanguo/shuguo.txt
```

### 2.3.3 下载

1）-copyToLocal：从 HDFS 拷贝到本地

```shell
hadoop fs -copyToLocal /sanguo/shuguo.txt ./
```

2）-get：等同于 copyToLocal，生产环境更习惯用 get

```shell
hadoop fs -get /sanguo/shuguo.txt ./shuguo2.txt
```

### 2.3.4 HDFS直接操作

1）-ls: 显示目录信息

```shell
hadoop fs -ls /sanguo
```

2）-cat：显示文件内容

```shell
hadoop fs -cat /sanguo/shuguo.txt
```

3）-chgrp、-chmod、-chown：Linux 文件系统中的用法一样，修改文件所属权限

```shell
hadoop fs  -chmod 666  /sanguo/shuguo.txt
hadoop fs  -chown  atguigu:atguigu   /sanguo/shuguo.txt
```

4）-mkdir：创建路径

```shell
hadoop fs -mkdir /jinguo
```

5）-cp：从 HDFS 的一个路径拷贝到 HDFS 的另一个路径

```shell
hadoop fs -cp /sanguo/shuguo.txt /jinguo
```

6）-mv：在 HDFS 目录中移动文件

```shell
hadoop fs -mv /sanguo/wuguo.txt /jinguo
hadoop fs -mv /sanguo/weiguo.txt /jinguo
```

7）-tail：显示一个文件的末尾 1kb 的数据

```shell
hadoop fs -tail /jinguo/shuguo.txt
```

8）-rm：删除文件或文件夹

```shell
hadoop fs -rm /sanguo/shuguo.txt
```

9）-rm -r：递归删除目录及目录里面内容

```shell
hadoop fs -rm -r /sanguo
```

10）-du：统计文件夹的大小信息

```shell
hadoop fs -du -s -h /jinguo

14  42  /jinguo/shuguo.txt
7   21   /jinguo/weiguo.txt
6   18   /jinguo/wuguo.tx
```

!!! info "说明"

    27 表示文件大小；81 表示 27 * 3 个副本；/jinguo 表示查看的目录

11）-setrep：设置 HDFS 中文件的副本数量

```shell
hadoop fs -setrep 10 
```

![image-20230226214630790](https://cos.gump.cloud/uPic/image-20230226214630790.png)

这里设置的副本数只是记录在 NameNode 的元数据中，是否真的会有这么多副本，还得看 DataNode 的数量。因为目前只有 3 台设备，最多也就 3 个副本，只有节点数的增加到 10 台时，副本数才能达到 10。
