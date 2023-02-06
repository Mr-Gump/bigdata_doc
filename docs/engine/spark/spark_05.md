# 第5章 累加器

累加器：分布式{++共享只写变量++}。（ Executor 和 Executor 之间不能读数据）。
累加器用来把 Executor 端变量信息聚合到 Driver 端。在 Driver 中定义的一个变量，在 Executor 端的每个 task 都会得到这个变量的一份新的副本，每个 task 更新这些副本的值后，传回 Driver 端进行合并计算。

![image-20230131163410816](https://cos.gump.cloud/uPic/image-20230131163410816.png)

1）累加器使用

累加器定义（SparkContext.accumulator(initialValue)方法）

LongAccumulator longAccumulator = JavaSparkContext.toSparkContext(sc).longAccumulator();	

累加器添加数据（累加器.add方法）

longAccumulator.add(stringIntegerTuple2._2);

累加器获取数据（累加器.value）

longAccumulator.value() 

2）创建包名：com.atguigu.accumulator

3）代码实现

```java
package com.atguigu.accumulator;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.VoidFunction;
import org.apache.spark.util.LongAccumulator;
import scala.Tuple2;

import java.util.Arrays;

public class Test01_Acc {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<String, Integer> tupleRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>("a", 1), new Tuple2<>("a", 2), new Tuple2<>("a", 3), new Tuple2<>("a", 1)));

        // 统计wordCount
        // 走shuffle 效率低
        JavaPairRDD<String, Integer> reduceByKeyRDD = tupleRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        // 使用变量无法实现
        /*
        final int[] i = {0};

        tupleRDD.foreach(new VoidFunction<Tuple2<String, Integer>>() {
            @Override
            public void call(Tuple2<String, Integer> stringIntegerTuple2) throws Exception {
                i[0] += stringIntegerTuple2._2;
            }
        });

        System.out.println(i[0]);
         */

        // 转换为scala的累加器
        LongAccumulator longAccumulator = JavaSparkContext.toSparkContext(sc).longAccumulator();

        // 在foreach中使用累加器
        tupleRDD.foreach(new VoidFunction<Tuple2<String, Integer>>() {
            @Override
            public void call(Tuple2<String, Integer> stringIntegerTuple2) throws Exception {
                longAccumulator.add(stringIntegerTuple2._2);
                //  不要在executor端获取累加器的值,因为不准确 因此我们说累加器叫分布式共享只写变量
                System.out.println(longAccumulator.value());
            }
        });

        System.out.println(longAccumulator.value());

        // 4. 关闭sc
        sc.stop();
    }
}
```

!!! warning "注意"

    Executor 端的任务不能读取累加器的值（例如：{++在 Executor 端调用 sum.value，获取的值不是累加器最终的值++}）。因此我们说，累加器是一个分布式共享只写变量。

!!! info "累加器要放在行动算子中"

    {++因为转换算子执行的次数取决于 job 的数量，如果一个 spark 应用有多个行动算子，那么转换算子中的累加器可能会发生不止一次更新，导致结果错误。++}所以，如果想要一个无论在失败还是重复计算时都绝对可靠的累加器，我们必须把它放在 foreach() 这样的行动算子中。

