# 第1章 HDFS概述

## 1.1 HDFS产出背景及定义

**1）HDFS 产生背景**

随着数据量越来越大，在一个操作系统存不下所有的数据，那么就分配到更多的操作系统管理的磁盘中，但是不方便管理和维护，迫切需要一种系统来{==管理多台机器上的文件==}，这就是分布式文件管理系统。{==HDFS 只是分布式文件管理系统中的一种==}。

**2）HDFS 定义**

HDFS（Hadoop Distributed File System），它是一个{==文件系统==}，用于存储文件，通过目录树来定位文件；其次，它是{==分布式==}的，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色。

HDFS 的使用场景：适合{==一次写入，多次读出==}的场景。一个文件经过创建、写入和关闭之后就不需要改变。

## 1.2 HDFS 优缺点

<figure markdown>
  ![image-20230226204917181](https://cos.gump.cloud/uPic/image-20230226204917181.png)
  <figcaption>HDFS优点</figcaption>
</figure>

<figure markdown>
  ![image-20230226204952256](https://cos.gump.cloud/uPic/image-20230226204952256.png)
  <figcaption>HDFS缺点</figcaption>
</figure>
## 1.3 HDFS组成架构

![image-20230226211615367](https://cos.gump.cloud/uPic/image-20230226211615367.png)

![image-20230226211715909](https://cos.gump.cloud/uPic/image-20230226211715909.png)

## 1.4 HDFS文件块大小（面试重点）

![image-20230226211754732](https://cos.gump.cloud/uPic/image-20230226211754732.png)

![image-20230226211830255](https://cos.gump.cloud/uPic/image-20230226211830255.png)

