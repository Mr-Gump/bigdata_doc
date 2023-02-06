# 第3章 RDD概述

## 3.1 什么是RDD

RDD（ Resilient Distributed Dataset ）叫做弹性分布式数据集，是 Spark 中最基本的数据抽象。



代码中是一个抽象类，它代表一个弹性的、不可变、可分区、里面的元素可并行计算的集合。



<figure markdown>
  ![image-20230131144702214](https://cos.gump.cloud/uPic/image-20230131144702214.png)
  <figcaption>RDD类比工厂生产</figcaption>
</figure>

## 3.2 RDD五大特性

![image-20230131144814170](https://cos.gump.cloud/uPic/image-20230131144814170.png)
