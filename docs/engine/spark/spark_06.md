# 第6章 广播变量

广播变量：分布式{++共享只读变量++}。
广播变量用来{==高效分发较大的对象==}。向所有工作节点发送一个较大的只读值，以供一个或多个 Spark Task 操作使用。比如，如果你的应用需要向所有节点发送一个较大的只读查询表，广播变量用起来会很顺手。在多个 Task 并行操作中使用同一个变量，但是 Spark 会为每个 Task 任务分别发送。

1）使用广播变量步骤：

（1）调用 SparkContext.broadcast（广播变量）创建出一个广播对象，任何可序列化的类型都可以这么实现。
（2）通过 广播变量.value，访问该对象的值。
（3）广播变量只会被发到各个节点一次，作为只读值处理（修改这个值不会影响到别的节点）。

2）原理说明

![image-20230131164220396](https://cos.gump.cloud/uPic/image-20230131164220396.png)

3）创建包名：com.atguigu.broadcast
4）代码实现

```java
package com.atguigu.accumulator;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.broadcast.Broadcast;

import java.util.Arrays;
import java.util.List;

public class Test02_Broadcast {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> intRDD = sc.parallelize(Arrays.asList(4, 56, 7, 8, 1, 2));

        // 幸运数字
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        // 找出幸运数字
        // 每一个task都会创建一个list浪费内存
        /*
        JavaRDD<Integer> result = intRDD.filter(new Function<Integer, Boolean>() {
            @Override
            public Boolean call(Integer v1) throws Exception {
                return list.contains(v1);
            }
        });
         */

        // 创建广播变量
        // 只发送一份数据到每一个executor
        Broadcast<List<Integer>> broadcast = sc.broadcast(list);

        JavaRDD<Integer> result = intRDD.filter(new Function<Integer, Boolean>() {
            @Override
            public Boolean call(Integer v1) throws Exception {
                return broadcast.value().contains(v1);
            }
        });

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

