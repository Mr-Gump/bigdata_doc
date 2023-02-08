# 第14章 Shuffle

Spark 最初版本 HashShuffle。
Spark 0.8.1 版本以后优化后的 HashShuffle。
Spark1.1 版本加入 SortShuffle，默认是 HashShuffle。
Spark1.2 版本默认是 SortShuffle，但是可配置 HashShuffle。
Spark2.0 版本删除 HashShuffle 只有 SortShuffle。

## 14.1 Shuffle的原理和执行过程

Shuffle 一定会有落盘。

- 如果 Shuffle 过程中落盘数据量减少，那么可以提高性能

- 算子如果存在预聚合功能，可以提高 Shuffle 的性能

![image-20230208193901444](https://cos.gump.cloud/uPic/image-20230208193901444.png)

## 14.2 HashShuffle解析

### 14.2.1 未优化的HashShuffle

![image-20230208193936443](https://cos.gump.cloud/uPic/image-20230208193936443.png)

### 14.2.2 优化后的HashShuffle

优化的 HashShuffle 过程就是启用合并机制，合并机制就是复用 buffer，开启合并机制的配置是 `spark.shuffle.consolidateFiles`。该参数默认值为 false，将其设置为 true 即可开启优化机制。通常来说，如果我们使用 HashShuffleManager，那么都建议开启这个选项。

官网参数说明：http://spark.apache.org/docs/0.8.1/configuration.html

![image-20230208194129588](https://cos.gump.cloud/uPic/image-20230208194129588.png)

## 14.3 SortShuffle解析

### 14.3.1 SortShuffle

![image-20230208194216889](https://cos.gump.cloud/uPic/image-20230208194216889.png)

在溢写磁盘前，先根据 key 进行排序，排序过后的数据，会分批写入到磁盘文件中。默认批次为 {==10000==} 条，数据会以每批一万条写入到磁盘文件。写入磁盘文件通过缓冲区溢写的方式，每次溢写都会产生一个磁盘文件，{=也=就是说一个 Task 过程会产生多个临时文件。最后在每个 Task 中，将所有的临时文件合并，这就是 merge 过程，此过程将所有临时文件读取出来，一次写入到最终文件==}。

### 14.3.2 bypassShuffle

bypassShuffle 和 SortShuffle 的区别就是不对数据排序。

bypass 运行机制的触发条件如下：

1）shuffle reduce task 数量小于等于 spark.shuffle.sort.bypassMergeThreshold 参数的值，默认为 200。

2）不是聚合类的 shuffle 算子（比如 reduceByKey 不行）。

![image-20230208194627016](https://cos.gump.cloud/uPic/image-20230208194627016.png)