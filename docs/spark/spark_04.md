# 第4章 RDD编程

## 4.1 RDD的创建

在 Spark 中 RDD 的创建方式可以分为三种：

1. 从集合中创建 RDD
2. 从外部存储创建 RDD
3. 从其他 RDD 创建

### 4.1.1 IDEA环境准备

1）创建一个 maven 工程，工程名称叫 SparkCore

![image-20230131145047409](https://cos.gump.cloud/uPic/image-20230131145047409.png)

2）创建包名：com.atguigu.createrdd

3）在 pom 文件中添加 spark-core 的依赖

```xml title="pom.xml"
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.12</artifactId>
        <version>3.1.3</version>
    </dependency>
</dependencies>
```

4）如果不希望运行时打印大量日志，可以在 resources 文件夹中添加 log4j.properties 文件，并添加日志配置信息

```properties title="log4j.properties"
log4j.rootCategory=ERROR, console
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n

# Set the default spark-shell log level to ERROR. When running the spark-shell, the
# log level for this class is used to overwrite the root logger's log level, so that
# the user can have different defaults for the shell and regular Spark apps.
log4j.logger.org.apache.spark.repl.Main=ERROR

# Settings to quiet third party logs that are too verbose
log4j.logger.org.spark_project.jetty=ERROR
log4j.logger.org.spark_project.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=ERROR
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=ERROR
log4j.logger.org.apache.parquet=ERROR
log4j.logger.parquet=ERROR

# SPARK-9183: Settings to avoid annoying messages when looking up nonexistent UDFs in SparkSQL with Hive support
log4j.logger.org.apache.hadoop.hive.metastore.RetryingHMSHandler=FATAL
log4j.logger.org.apache.hadoop.hive.ql.exec.FunctionRegistry=ERROR
```

### 4.1.2 创建IDEA快捷键

1）点击File -> Settings… -> Editor -> Live Templates -> output -> Live Template

![image-20230131145256397](https://cos.gump.cloud/uPic/image-20230131145256397.png)

![image-20230131145302701](https://cos.gump.cloud/uPic/image-20230131145302701.png)

2）点击左下角的 Define -> 选择 JAVA

![image-20230131145320182](https://cos.gump.cloud/uPic/image-20230131145320182.png)

3）在 Abbreviation 中输入快捷键名称 sc，在 Template text 中填写，输入快捷键后生成的内容。

![image-20230131145339328](https://cos.gump.cloud/uPic/image-20230131145339328.png)

```java
// 1.创建配置对象
SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

// 2. 创建sparkContext
JavaSparkContext sc = new JavaSparkContext(conf);

// 3. 编写代码

// 4. 关闭sc
sc.stop();
```

### 4.1.3 异常处理

如果本机操作系统是 Windows，如果在程序中使用了 Hadoop 相关的东西，比如写入文件到 HDFS，则会遇到如下异常：

![image-20230131145439765](https://cos.gump.cloud/uPic/image-20230131145439765.png)

出现这个问题的原因，并不是程序的错误，而是用到了 Hadoop 相关的服务，解决办法

1）配置 HADOOP_HOME 环境变量
2）在 IDEA 中配置 Run Configuration，添加 HADOOP_HOME 变量

![image-20230131145518065](https://cos.gump.cloud/uPic/image-20230131145518065.png)

![image-20230131145523497](https://cos.gump.cloud/uPic/image-20230131145523497.png)

如果出现这个问题，是因为 windows 上面的 hadoop 权限不够。

```bash
Exception in thread "main" java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Native Method)
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access(NativeIO.java:645)
	at org.apache.hadoop.fs.FileUtil.canRead(FileUtil.java:1230)
	at org.apache.hadoop.fs.FileUtil.list(FileUtil.java:1435)
	at org.apache.hadoop.fs.RawLocalFileSystem.listStatus(RawLocalFileSystem.java:493)
	at org.apache.hadoop.fs.FileSystem.listStatus(FileSystem.java:1868)
	at org.apache.hadoop.fs.FileSystem.listStatus(FileSystem.java:1910)
	at org.apache.hadoop.fs.FileSystem$4.<init>(FileSystem.java:2072)
```

解决方法是把安装在 windows 上面的 hadoop 的 bin 文件夹中的 hadoop.dll 复制到 C:\Windows\System32 文件夹中。

![image-20230131145615506](https://cos.gump.cloud/uPic/image-20230131145615506.png)

![image-20230131145619226](https://cos.gump.cloud/uPic/image-20230131145619226.png)

### 4.1.4 从集合中创建

1）从集合中创建 RDD：parallelize

```java
package com.atguigu.createrdd;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;
import java.util.List;

public class Test01_List {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> stringRDD = sc.parallelize(Arrays.asList("hello", "spark"));

        List<String> result = stringRDD.collect();

        for (String s : result) {
            System.out.println(s);
        }

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.1.5 从外部存储系统的数据集创建

由外部存储系统的数据集创建 RDD 包括：本地的文件系统，还有所有 Hadoop 支持的数据集，比如 HDFS、HBase 等。

1）数据准备

在新建的 SparkCore 项目名称上右键 =》 新建 input 文件夹 =》 在 input 文件夹上右键 =》 分别新建 1.txt 和 2.txt 。每个文件里面准备一些 word 单词。



2）创建RDD

```java
package com.atguigu.createrdd;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.List;

public class Test02_File {
    public static void main(String[] args) {
        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input");

        List<String> result = lineRDD.collect();

        for (String s : result) {
            System.out.println(s);
        }

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.1.6 从其他RDD创建

主要是通过一个 RDD 运算完后，再产生新的 RDD。

详见[第三小节](#43-transformation)

## 4.2 分区规则

### 4.2.1 从集合创建RDD

1）创建一个包名： com.atguigu.partition
2）代码验证

```java
package com.atguigu.partition;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;

public class Test01_ListPartition {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 默认环境的核数
        // 可以手动填写参数控制分区的个数
        JavaRDD<String> stringRDD = sc.parallelize(Arrays.asList("hello", "spark", "hello", "spark", "hello"),2);

        // 数据分区的情况
        // 0 => 1,2  1 => 3,4,5
        // 利用整数除机制  左闭右开
        // 0 => start 0*5/2  end 1*5/2
        // 1 => start 1*5/2  end 2*5/2
        stringRDD.saveAsTextFile("output");

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.2.2 从文件创建RDD

1）分区测试

```java
package com.atguigu.partition;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.List;

public class Test02_FilePartition {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 默认填写的最小分区数   2和环境的核数取小的值  一般为2
        JavaRDD<String> lineRDD = sc.textFile("input/1.txt");

        // 具体的分区个数需要经过公式计算
        // 首先获取文件的总长度  totalSize
        // 计算平均长度  goalSize = totalSize / numSplits
        // 获取块大小 128M
        // 计算切分大小  splitSize = Math.max(minSize, Math.min(goalSize, blockSize));
        // 最后使用splitSize  按照1.1倍原则切分整个文件   得到几个分区就是几个分区

        // 实际开发中   只需要看文件总大小 / 填写的分区数  和块大小比较  谁小拿谁进行切分
        lineRDD.saveAsTextFile("output");

        // 数据会分配到哪个分区
        // 如果切分的位置位于一行的中间  会在当前分区读完一整行数据

        // 0 -> 1,2  1 -> 3  2 -> 4  3 -> 空

        // 4. 关闭sc
        sc.stop();
    }
}
```

2）分区源码

!!! info "注意"

    getSplits 文件返回的是切片规划，真正读取是在 compute 方法中创建 LineRecordReader 读取的，有两个关键变量： `start = split.getStart()`,`  end = start + split.getLength`
    
    ①分区数量的计算方式:
    totalSize = 10
    goalSize = 10 / 3 = 3(byte) 表示每个分区存储 3 字节的数据
    分区数 = totalSize / goalSize = 10 /3 => 3,3,4
    4 字节大于 3 子节的 1.1 倍,符合 hadoop 切片 1.1 倍的策略,因此会多创建一个分区,即一共有 4 个分区  3,3,3,1
    
    ②Spark读取文件，采用的是hadoop的方式读取，所以一行一行读取，跟字节数没有关系
    
    ③数据读取位置计算是以偏移量为单位来进行计算的。

## 4.3 Transformation转换算子

### 4.3.1 Value类型

1）创建包名： com.atguigu.value

#### 4.3.1.1 map()映射

参数 f 是一个函数，可以写作匿名子类，它可以接收一个参数。当某个 RDD 执行 map 方法时，会遍历该 RDD 中的每一个数据项，并依次应用 f 函数，从而产生一个新的 RDD。即，这个新 RDD 中的每一个元素都是原来 RDD	 中每一个元素依次应用 f 函数而得到的。

1）具体实现

```java
package com.atguigu.value;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

public class Test01_Map {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/1.txt");

        // 需求:每行结尾拼接||
        // 两种写法  lambda表达式写法(匿名函数) 
        JavaRDD<String> mapRDD = lineRDD.map(s -> s + "||");

        // 匿名函数写法 
        JavaRDD<String> mapRDD1 = lineRDD.map(new Function<String, String>() {
            @Override
            public String call(String v1) throws Exception {
                return v1 + "||";
            }
        });

        for (String s : mapRDD.collect()) {
            System.out.println(s);
        }

        // 输出数据的函数写法
        mapRDD1.collect().forEach(a -> System.out.println(a));
        mapRDD1.collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.1.2 flatMap()扁平化

1）功能说明

与 map 操作类似，将 RDD 中的每一个元素通过应用 f 函数依次转换为新的元素，并封装到 RDD 中。
区别：在 flatMap 操作中，f 函数的返回值是一个集合，并且会将每一个该集合中的元素拆分出来放到新的 RDD 中。

2）需求说明

创建一个集合，集合里面存储的还是子集合，把所有子集合中数据取出放入到一个大的集合中。

![image-20230131151150529](https://cos.gump.cloud/uPic/image-20230131151150529.png)

4）具体实现：

```java
package com.atguigu.value;

import org.apache.commons.collections.ListUtils;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class Test02_FlatMap {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        ArrayList<List<String>> arrayLists = new ArrayList<>();

        arrayLists.add(Arrays.asList("1","2","3"));
        arrayLists.add(Arrays.asList("4","5","6"));


        JavaRDD<List<String>> listJavaRDD = sc.parallelize(arrayLists,2);
        // 对于集合嵌套的RDD 可以将元素打散
        // 泛型为打散之后的元素类型
        JavaRDD<String> stringJavaRDD = listJavaRDD.flatMap(new FlatMapFunction<List<String>, String>() {
            @Override
            public Iterator<String> call(List<String> strings) throws Exception {

                return strings.iterator();
            }
        });

        stringJavaRDD. collect().forEach(System.out::println);

        // 通常情况下需要自己将元素转换为集合
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");

        JavaRDD<String> stringJavaRDD1 = lineRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                String[] s1 = s.split(" ");
                return Arrays.asList(s1).iterator();
            }
        });

        stringJavaRDD1. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.1.3 groupBy()分组

1）功能说明：分组，按照传入函数的返回值进行分组。将相同的 key 对应的值放入一个迭代器。
2）需求说明：创建一个 RDD，按照元素模以 2 的值进行分组。
3）具体实现

```java
package com.atguigu.value;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

import java.util.Arrays;

public class Test03_GroupBy {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        // 泛型为分组标记的类型
        JavaPairRDD<Integer, Iterable<Integer>> groupByRDD = integerJavaRDD.groupBy(new Function<Integer, Integer>() {
            @Override
            public Integer call(Integer v1) throws Exception {
                return v1 % 2;
            }
        });

        groupByRDD.collect().forEach(System.out::println);

        // 类型可以任意修改
        JavaPairRDD<Boolean, Iterable<Integer>> groupByRDD1 = integerJavaRDD.groupBy(new Function<Integer, Boolean>() {
            @Override
            public Boolean call(Integer v1) throws Exception {
                return v1 % 2 == 0;
            }
        });

        groupByRDD1. collect().forEach(System.out::println);

	Thread.sleep(600000);

        // 4. 关闭sc
        sc.stop();
    }
}
```

- groupBy 会存在 shuffle 过程
- shuffle：将不同的分区数据进行打乱重组的过程
- shuffle 一定会落盘。可以在 local 模式下执行程序，通过 4040 看效果

#### 4.3.1.4 filter()过滤

1）功能说明
接收一个返回值为布尔类型的函数作为参数。当某个 RDD 调用 filter 方法时，会对该 RDD 中每一个元素应用 f 函数，如果返回值类型为 true，则该元素会被添加到新的 RDD 中。

2）需求说明：创建一个 RDD，过滤出对 2 取余等于 0 的数据

![image-20230131151421821](https://cos.gump.cloud/uPic/image-20230131151421821.png)

3）代码实现

```java
package com.atguigu.value;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

import java.util.Arrays;

public class Test04_Filter {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        JavaRDD<Integer> filterRDD = integerJavaRDD.filter(new Function<Integer, Boolean>() {
            @Override
            public Boolean call(Integer v1) throws Exception {
                return v1 % 2 == 0;
            }
        });

        filterRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.3.1.5 distinct()去重

1）功能说明：对内部的元素去重，并将去重后的元素放到新的 RDD 中。
2）代码实现

```java
package com.atguigu.value;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;


public class Test05_Distinct {
    public static void main(String[] args) {
        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4, 5, 6), 2);

        // 底层使用分布式分组去重  所有速度比较慢,但是不会OOM
        JavaRDD<Integer> distinct = integerJavaRDD.distinct();

        distinct. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

!!! info "注意"
    distinct 会存在 shuffle 过程。
    

#### 4.3.1.6 sortBy()排序

1）功能说明

该操作用于排序数据。在排序之前，可以将数据通过 f 函数进行处理，之后按照 f 函数处理的结果进行排序，默认为正序排列。排序后新产生的 RDD 的分区数与原 RDD 的分区数一致。Spark 的排序结果是全局有序。

2）需求说明：创建一个 RDD，按照数字大小分别实现正序和倒序排序

![image-20230131151643837](https://cos.gump.cloud/uPic/image-20230131151643837.png)

3）代码实现：

```java
package com.atguigu.value;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

import java.util.Arrays;

public class Test6_SortBy {
    public static void main(String[] args) {
        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(5, 8, 1, 11, 20), 2);

        // (1)泛型为以谁作为标准排序  (2) true为正序  (3) 排序之后的分区个数
        JavaRDD<Integer> sortByRDD = integerJavaRDD.sortBy(new Function<Integer, Integer>() {
            @Override
            public Integer call(Integer v1) throws Exception {
                return v1;
            }
        }, true, 2);

        sortByRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.2 Key-Value类型

1）创建包名： com.atguigu.keyvalue

要想使用 Key-Value 类型的算子首先需要使用特定的方法转换为 PairRDD

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;

public class Test01_pairRDD{
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);


        JavaPairRDD<Integer, Integer> pairRDD = integerJavaRDD.mapToPair(new PairFunction<Integer, Integer, Integer>() {
            @Override
            public Tuple2<Integer, Integer> call(Integer integer) throws Exception {
                return new Tuple2<>(integer, integer);
            }
        });

        pairRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.2.1 mapValues()只对V进行操作

1）功能说明：针对于 (K,V) 形式的类型只对 V 进行操作
2）需求说明：创建一个 pairRDD，并将 value 添加字符串"|||"

![image-20230131151815113](https://cos.gump.cloud/uPic/image-20230131151815113.png)

4）代码实现：

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import scala.Tuple2;

import java.util.Arrays;

public class Test02_MapValues {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<String, String> javaPairRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>("k", "v"), new Tuple2<>("k1", "v1"), new Tuple2<>("k2", "v2")));

        // 只修改value 不修改key
        JavaPairRDD<String, String> mapValuesRDD = javaPairRDD.mapValues(new Function<String, String>() {
            @Override
            public String call(String v1) throws Exception {
                return v1 + "|||";
            }
        });

        mapValuesRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.2.2 groupByKey()按照K重新分组

1）功能说明

groupByKey 对每个 key 进行操作，但只生成一个 seq，并不进行聚合。

该操作可以指定分区器或者分区数（默认使用 HashPartitioner ）

2）需求说明：统计单词出现次数

![image-20230131151904009](https://cos.gump.cloud/uPic/image-20230131151904009.png)

4）代码实现：

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;

public class Test03_GroupByKey {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> integerJavaRDD = sc.parallelize(Arrays.asList("hi","hi","hello","spark" ),2);

        // 统计单词出现次数
        JavaPairRDD<String, Integer> pairRDD = integerJavaRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });

        // 聚合相同的key
        JavaPairRDD<String, Iterable<Integer>> groupByKeyRDD = pairRDD.groupByKey();

        // 合并值
        JavaPairRDD<String, Integer> result = groupByKeyRDD.mapValues(new Function<Iterable<Integer>, Integer>() {
            @Override
            public Integer call(Iterable<Integer> v1) throws Exception {
                Integer sum = 0;
                for (Integer integer : v1) {
                    sum += integer;
                }
                return sum;
            }
        });

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}}
```

#### 4.3.2.3 reduceByKey()按照K聚合V

1）功能说明：该操作可以将 RDD[K,V] 中的元素按照相同的 K 对 V 进行聚合。其存在多种重载形式，还可以设置新 RDD 的分区数。
2）需求说明：统计单词出现次数

![image-20230131152025663](https://cos.gump.cloud/uPic/image-20230131152025663.png)

3）代码实现：

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;

public class Test04_ReduceByKey {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> integerJavaRDD = sc.parallelize(Arrays.asList("hi","hi","hello","spark" ),2);

        // 统计单词出现次数
        JavaPairRDD<String, Integer> pairRDD = integerJavaRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });

        // 聚合相同的key
        JavaPairRDD<String, Integer> result = pairRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.2.4 reduceByKey和groupByKey区别

1）reduceByKey：按照 key 进行聚合，在 shuffle 之前有 combine（预聚合）操作，返回结果是 RDD[K,V]。
2）groupByKey：按照 key 进行分组，直接进行 shuffle。
3）开发指导：在不影响业务逻辑的前提下，优先选用 reduceByKey。求和操作不影响业务逻辑，求平均值影响业务逻辑。影响业务逻辑时建议先对数据类型进行转换再合并。

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;

public class Test06_ReduceByKeyAvg {
    public static void main(String[] args) throws InterruptedException {
        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<String, Integer> javaPairRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>("hi", 96), new Tuple2<>("hi", 97), new Tuple2<>("hello", 95), new Tuple2<>("hello", 195)));

        // ("hi",(96,1))
        JavaPairRDD<String, Tuple2<Integer, Integer>> tuple2JavaPairRDD = javaPairRDD.mapValues(new Function<Integer, Tuple2<Integer, Integer>>() {
            @Override
            public Tuple2<Integer, Integer> call(Integer v1) throws Exception {
                return new Tuple2<>(v1, 1);
            }
        });

        // 聚合RDD
        JavaPairRDD<String, Tuple2<Integer, Integer>> reduceRDD = tuple2JavaPairRDD.reduceByKey(new Function2<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>, Tuple2<Integer, Integer>>() {
            @Override
            public Tuple2<Integer, Integer> call(Tuple2<Integer, Integer> v1, Tuple2<Integer, Integer> v2) throws Exception {
                return new Tuple2<>(v1._1 + v2._1, v1._2 + v2._2);
            }
        });

        // 相除
        JavaPairRDD<String, Double> result = reduceRDD.mapValues(new Function<Tuple2<Integer, Integer>, Double>() {
            @Override
            public Double call(Tuple2<Integer, Integer> v1) throws Exception {
                return (new Double(v1._1) / v1._2);
            }
        });

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

#### 4.3.2.5 sortByKey()按照K进行排序

1）功能说明
在一个 (K,V) 的 RDD 上调用，K 必须实现 Ordered 接口，返回一个按照 key 进行排序的 (K,V) 的 RDD。
2）需求说明：创建一个 pairRDD，按照 key 的正序和倒序进行排序

![image-20230131152237681](https://cos.gump.cloud/uPic/image-20230131152237681.png)

3）代码实现：

```java
package com.atguigu.keyValue;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.util.Arrays;

public class Test05_SortByKey {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<Integer, String> javaPairRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>(4, "a"), new Tuple2<>(3, "c"), new Tuple2<>(2, "d")));

        // 填写布尔类型选择正序倒序
        JavaPairRDD<Integer, String> pairRDD = javaPairRDD.sortByKey(false);

        pairRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

## 4.4 Action行动算子

行动算子是触发了整个作业的执行。因为转换算子都是懒加载，并不会立即执行。
1）创建包名： com.atguigu.action

### 4.4.1 collect()以数组的形式返回数据集

1）功能说明：在 Driver 中，以数组 Array 的形式返回数据集的所有元素。

![image-20230131152335962](https://cos.gump.cloud/uPic/image-20230131152335962.png)

!!! warning "注意"

    所有的数据都会被拉取到Driver端，慎用。

2）需求说明：创建一个 RDD，并将 RDD 内容收集到 Driver 端打印

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;
import java.util.List;

public class Test01_Collect {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        List<Integer> collect = integerJavaRDD.collect();

        for (Integer integer : collect) {
            System.out.println(integer);
        }

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.2 count()返回RDD中元素个数

1）功能说明：返回 RDD 中元素的个数

![image-20230131152455791](https://cos.gump.cloud/uPic/image-20230131152455791.png)

3）需求说明：创建一个 RDD，统计该 RDD 的数据条数

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;


public class Test02_Count {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        long count = integerJavaRDD.count();

        System.out.println(count);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.3 first()返回RDD中的第一个元素

1）功能说明：返回 RDD 中的第一个元素

![image-20230131152542425](https://cos.gump.cloud/uPic/image-20230131152542425.png)

2）需求说明：创建一个 RDD，返回该 RDD 中的第一个元素

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;

public class Test03_First {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        Integer first = integerJavaRDD.first();

        System.out.println(first);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.4 take()返回由RDD前n个元素组成的数组

1）功能说明：返回一个由 RDD 的前 n 个元素组成的数组

![image-20230131152625672](https://cos.gump.cloud/uPic/image-20230131152625672.png)

2）需求说明：创建一个 RDD，取出前两个元素

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;

import java.util.Arrays;
import java.util.List;

public class Test04_Take {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        List<Integer> list = integerJavaRDD.take(3);

        list.forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.5 countByKey()统计每种key的个数

1）功能说明：统计每种 key 的个数

![image-20230131152707311](https://cos.gump.cloud/uPic/image-20230131152707311.png)

2）需求说明：创建一个 PairRDD，统计每种 key 的个数

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Map;

public class Test05_CountByKey {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<String, Integer> pairRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>("a", 8), new Tuple2<>("b", 8), new Tuple2<>("a", 8), new Tuple2<>("d", 8)));

        Map<String, Long> map = pairRDD.countByKey();

        System.out.println(map);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.6 save相关算子

1）`saveAsTextFile(path)` 保存成 Text 文件
功能说明：将数据集的元素以 textfile 的形式保存到 HDFS 文件系统或者其他支持的文件系统，对于每个元素， Spark 将会调用 toString 方法，将它转换为文件中的文本

2）`saveAsObjectFile(path)` 序列化成对象保存到文件
功能说明：用于将 RDD 中的元素序列化成对象，存储到文件中。

3）代码实现

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.util.Arrays;

public class Test06_Save {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),2);

        integerJavaRDD.saveAsTextFile("output");

        integerJavaRDD.saveAsObjectFile("output1");


        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.7 foreach()遍历RDD中每一个元素

![image-20230131152932978](https://cos.gump.cloud/uPic/image-20230131152932978.png)

2）需求说明：创建一个 RDD，对每个元素进行打印

```java
package com.atguigu.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.VoidFunction;

import java.util.Arrays;

public class Test07_Foreach {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> integerJavaRDD = sc.parallelize(Arrays.asList(1, 2, 3, 4),4);

        integerJavaRDD.foreach(new VoidFunction<Integer>() {
            @Override
            public void call(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.4.8 foreachPartition ()遍历RDD中每一个分区

```java
package com.atguigu.spark.action;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.VoidFunction;

import java.util.Arrays;
import java.util.Iterator;

public class Test08_ForeachPartition {
    public static void main(String[] args) {

        // 1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("core").setMaster("local[*]");

        // 2. 创建sc环境
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<Integer> parallelize = sc.parallelize(Arrays.asList(1, 2, 3, 4, 5, 6), 2);

        // 多线程一起计算   分区间无序  单个分区有序
        parallelize.foreachPartition(new VoidFunction<Iterator<Integer>>() {
            @Override
            public void call(Iterator<Integer> integerIterator) throws Exception {
                // 一次处理一个分区的数据
                while (integerIterator.hasNext()) {
                    Integer next = integerIterator.next();
                    System.out.println(next);
                }
            }
        });

        // 4. 关闭sc
        sc.stop();
    }
}
```

## 2.5 WordCount案例实操

1）导入项目依赖

```xml title="pom.xml"
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.12</artifactId>
        <version>3.1.3</version>
    </dependency>
</dependencies>
```

### 2.5.1 本地调试

本地 Spark 程序调试需要使用 Local 提交模式，即将本机当做运行环境，Master 和 Worker 都为本机。运行时直接加断点调试即可。如下：
1）代码实现

```java
package com.atguigu;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class WordCount {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 读取数据
        JavaRDD<String> javaRDD = sc.textFile("input/2.txt");

        // 长字符串切分为单个单词
        JavaRDD<String> flatMapRDD = javaRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                List<String> strings = Arrays.asList(s.split(" "));
                return strings.iterator();
            }
        });

        // 转换格式为  (单词,1)
        JavaPairRDD<String, Integer> pairRDD = flatMapRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });

        // 合并相同单词
        JavaPairRDD<String, Integer> javaPairRDD = pairRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        javaPairRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

2）调试流程

![image-20230131153209726](https://cos.gump.cloud/uPic/image-20230131153209726.png)

### 4.5.2 集群运行

1）修改代码，修改运行模式，将输出的方法修改为落盘，同时设置可以自定义的传入传出路径

```java
package com.atguigu;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class WordCount {
    public static void main(String[] args) {
        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("yarn").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 读取数据
        JavaRDD<String> javaRDD = sc.textFile(args[0]);

        // 长字符串切分为单个单词
        JavaRDD<String> flatMapRDD = javaRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                List<String> strings = Arrays.asList(s.split(" "));
                return strings.iterator();
            }
        });

        // 转换格式为  (单词,1)
        JavaPairRDD<String, Integer> pairRDD = flatMapRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });

        // 合并相同单词
        JavaPairRDD<String, Integer> javaPairRDD = pairRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        javaPairRDD.saveAsTextFile(args[1]);

        // 4. 关闭sc
        sc.stop();
    }
}
```

![image-20230131153255651](https://cos.gump.cloud/uPic/image-20230131153255651.png)

（2）将 WordCount.jar 上传到 /opt/module/spark-yarn 目录

（3）在 HDFS 上创建，存储输入文件的路径 /input

```bash
 hadoop fs -mkdir /input
```

（4）上传输入文件到 /input 路径

```bash
hadoop fs -put /opt/software /input
```

（5）执行任务

```bash
bin/spark-submit \
--class com.atguigu.spark.WordCount \
--master yarn \
./WordCount.jar \
/input \
/output
```

注意：input 和 ouput 都是 HDFS 上的集群路径。

（6）查询运行结果

```bash
hadoop fs -cat /output/*
```

## 4.6 RDD序列化

在实际开发中我们往往需要自己定义一些对于 RDD 的操作，那么此时需要注意的是，初始化工作是在 Driver 端进行的，而实际运行程序是在 Executor 端进行的，这就涉及到了跨进程通信，是需要序列化的。下面我们看几个例子：

### 4.6.1 序列化异常

0）创建包名： com.atguigu.serializable
1）创建使用的 javaBean：User

```java
package com.atguigu.bean;

import java.io.Serializable;

public class User implements Serializable {
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public User() {
    }


    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

2）创建类：Test01_Ser 测试序列化

```java
package com.atguigu.serializable;

import com.atguigu.bean.User;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

import java.util.Arrays;

public class Test01_Ser {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        User zhangsan = new User("zhangsan", 13);
        User lisi = new User("lisi", 13);

        JavaRDD<User> userJavaRDD = sc.parallelize(Arrays.asList(zhangsan, lisi), 2);

        JavaRDD<User> mapRDD = userJavaRDD.map(new Function<User, User>() {
            @Override
            public User call(User v1) throws Exception {
                return new User(v1.getName(), v1.getAge() + 1);
            }
        });

        mapRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.6.2 Kryo序列化框架

[:material-link:参考地址]( https://github.com/EsotericSoftware/kryo)

Java 的序列化能够序列化任何类。但是比较重，序列化后对象的体积也比较大。

Spark 出于性能的考虑，Spark2.0 开始支持另外一种 Kryo 序列化机制。Kryo 速度是 Serializable 的 10 倍。当 RDD 在 Shuffle 数据的时候，简单数据类型、数组和字符串类型已经在 Spark 内部使用 Kryo 序列化。

```java
package com.atguigu.serializable;

import com.atguigu.bean.User;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;

import java.util.Arrays;

public class Test02_Kryo {
    public static void main(String[] args) throws ClassNotFoundException {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore")
                // 替换默认的序列化机制
                .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
                // 注册需要使用kryo序列化的自定义类
                .registerKryoClasses(new Class[]{Class.forName("com.atguigu.bean.User")});

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        User zhangsan = new User("zhangsan", 13);
        User lisi = new User("lisi", 13);

        JavaRDD<User> userJavaRDD = sc.parallelize(Arrays.asList(zhangsan, lisi), 2);

        JavaRDD<User> mapRDD = userJavaRDD.map(new Function<User, User>() {
            @Override
            public User call(User v1) throws Exception {
                return new User(v1.getName(), v1.getAge() + 1);
            }
        });

        mapRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

## 4.7 RDD依赖关系

### 4.7.1 查看血缘关系

RDD 只支持粗粒度转换，即在大量记录上执行的单个操作。将创建 RDD 的一系列 Lineage（血统）记录下来，以便恢复丢失的分区。RDD 的 Lineage 会记录 RDD 的元数据信息和转换行为，当该 RDD 的部分分区数据丢失时，它可以根据这些信息来重新运算和恢复丢失的数据分区。

![image-20230131153907492](https://cos.gump.cloud/uPic/image-20230131153907492.png)

!!! info "注意"

    通过 `toDebugString` 方法，查看 RDD 血缘关系

0）创建包名： com.atguigu.dependency

1）代码实现

```java
package com.atguigu.dependency;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class Test01_Dep {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");
        System.out.println(lineRDD.toDebugString());
        System.out.println("-------------------");

        JavaRDD<String> wordRDD = lineRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                List<String> stringList = Arrays.asList(s.split(" "));
                return stringList.iterator();
            }
        });
        System.out.println(wordRDD.toDebugString());
        System.out.println("-------------------");

        JavaPairRDD<String, Integer> tupleRDD = wordRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });
        System.out.println(tupleRDD.toDebugString());
        System.out.println("-------------------");

        JavaPairRDD<String, Integer> wordCountRDD = tupleRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });
        System.out.println(wordCountRDD.toDebugString());
        System.out.println("-------------------");
        
        // 4. 关闭sc
        sc.stop();
    }
}
```

2）打印结果

```bash
(2) input/2.txt MapPartitionsRDD[1] at textFile at Test01_Dep.java:29 []
 |  input/2.txt HadoopRDD[0] at textFile at Test01_Dep.java:29 []
(2) MapPartitionsRDD[2] at flatMap at Test01_Dep.java:32 []
 |  input/2.txt MapPartitionsRDD[1] at textFile at Test01_Dep.java:29 []
 |  input/2.txt HadoopRDD[0] at textFile at Test01_Dep.java:29 []
(2) MapPartitionsRDD[3] at mapToPair at Test01_Dep.java:42 []
 |  MapPartitionsRDD[2] at flatMap at Test01_Dep.java:32 []
 |  input/2.txt MapPartitionsRDD[1] at textFile at Test01_Dep.java:29 []
 |  input/2.txt HadoopRDD[0] at textFile at Test01_Dep.java:29 []
(2) ShuffledRDD[4] at reduceByKey at Test01_Dep.java:50 []
 +-(2) MapPartitionsRDD[3] at mapToPair at Test01_Dep.java:42 []
    |  MapPartitionsRDD[2] at flatMap at Test01_Dep.java:32 []
    |  input/2.txt MapPartitionsRDD[1] at textFile at Test01_Dep.java:29 []
    |  input/2.txt HadoopRDD[0] at textFile at Test01_Dep.java:29 []
```

!!! info "注意"

    圆括号中的数字表示 RDD 的并行度，也就是有几个分区

### 4.7.2 窄依赖

窄依赖表示{++每一个父 RDD 的 Partition 最多被子 RDD 的一个 Partition 使用（一对一or多对一）++}，窄依赖我们形象的比喻为独生子女。

![image-20230131154220700](https://cos.gump.cloud/uPic/image-20230131154220700.png)

### 4.7.3 宽依赖

宽依赖表示{++同一个父 RDD 的 Partition 被多个子 RDD 的 Partition 依赖（只能是一对多）++}，会引起 Shuffle，总结：宽依赖我们形象的比喻为超生。

![image-20230131154319642](https://cos.gump.cloud/uPic/image-20230131154319642.png)

具有宽依赖的 transformations 包括：sort、reduceByKey、groupByKey、join 和调用 rePartition 函数的任何操作。
宽依赖对 Spark 去评估一个 transformations 有更加重要的影响，比如对性能的影响。
在不影响业务要求的情况下，要尽量避免使用有宽依赖的转换算子，因为有宽依赖，就一定会走 shuffle，影响性能。

### 4.7.4 Stage任务划分

1）DAG 有向无环图
DAG（Directed Acyclic Graph）有向无环图是由点和线组成的拓扑图形，该图形具有方向，不会闭环。例如， DAG 记录了 RDD 的转换过程和任务的阶段。

![image-20230131154431921](https://cos.gump.cloud/uPic/image-20230131154431921.png)

2）任务运行的整体流程

<figure markdown>
  ![image-20230131154503256](https://cos.gump.cloud/uPic/image-20230131154503256.png)
  <figcaption>YarnClient运行模式</figcaption>
</figure>
![image-20230131154813302](https://cos.gump.cloud/uPic/image-20230131154813302.png)

3）RDD任务切分中间分为：Application、Job、Stage和Task

- Application：初始化一个 SparkContext 即生成一个 Application；
- Job：一个 Action 算子就会生成一个 Job；
- Stage：Stage 等于宽依赖的个数加 1；
- Task：一个 Stage 阶段中，最后一个 RDD 的分区个数就是 Task 的个数。



!!! info "注意"

    Application -> Job -> Stage -> Task 每一层都是 1 对 n 的关系。

![image-20230131155305570](https://cos.gump.cloud/uPic/image-20230131155305570.png)

4）代码实现

```java
package com.atguigu.dependency;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class Test02_Dep {
    public static void main(String[] args) throws InterruptedException {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");
        System.out.println(lineRDD.toDebugString());
        System.out.println("-------------------");

        JavaRDD<String> wordRDD = lineRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                List<String> stringList = Arrays.asList(s.split(" "));
                return stringList.iterator();
            }
        });
        System.out.println(wordRDD);
        System.out.println("-------------------");

        JavaPairRDD<String, Integer> tupleRDD = wordRDD.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<>(s, 1);
            }
        });
        System.out.println(tupleRDD.toDebugString());
        System.out.println("-------------------");

        // 缩减分区
        JavaPairRDD<String, Integer> coalesceRDD = tupleRDD.coalesce(1);

        JavaPairRDD<String, Integer> wordCountRDD = coalesceRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        },4);
        System.out.println(wordCountRDD.toDebugString());
        System.out.println("-------------------");

        wordCountRDD. collect().forEach(System.out::println);
wordCountRDD. collect().forEach(System.out::println);

        Thread.sleep(600000);

        // 4. 关闭sc
        sc.stop();
    }
}
```

5）查看 Job 个数

查看 http://localhost:4040/jobs/，发现 Job 有两个。

![image-20230131155556545](https://cos.gump.cloud/uPic/image-20230131155556545.png)

6）查看 Stage 个数

查看 Job0 的 Stage。由于只有 1 个 Shuffle 阶段，所以 Stage 个数为 2。

![image-20230131155626366](https://cos.gump.cloud/uPic/image-20230131155626366.png)

![image-20230131155635855](https://cos.gump.cloud/uPic/image-20230131155635855.png)

查看 Job1 的 Stage。由于只有 1 个 Shuffle 阶段，所以 Stage 个数为 2。

![image-20230131155657920](https://cos.gump.cloud/uPic/image-20230131155657920.png)

![image-20230131155704150](https://cos.gump.cloud/uPic/image-20230131155704150.png)

7）Task 个数

查看 Job0 的 Stage0 的 Task 个数，2 个。

![image-20230131155746988](https://cos.gump.cloud/uPic/image-20230131155746988.png)

查看 Job0 的 Stage1 的 Task 个数，2 个。

![image-20230131155802885](https://cos.gump.cloud/uPic/image-20230131155802885.png)

![image-20230131155807068](https://cos.gump.cloud/uPic/image-20230131155807068.png)

查看 Job1 的 Stage2 的 Task 个数，0个（2个跳过skipped）。

![image-20230131155825435](https://cos.gump.cloud/uPic/image-20230131155825435.png)

查看 Job1 的 Stage3 的 Task 个数，2 个。

![image-20230131155842050](https://cos.gump.cloud/uPic/image-20230131155842050.png)

!!! info "注意" 

    如果存在 shuffle 过程，系统会自动进行缓存，UI 界面显示 skipped 的部分。

## 4.8 RDD持久化

## 4.8.1 RDD Cache缓存

RDD 通过 Cache 或者 Persist 方法将前面的计算结果缓存，默认情况下会把数据以序列化的形式缓存在 JVM 的堆内存中。但是并不是这两个方法被调用时立即缓存，而是触发后面的 action 算子时，该 RDD 将会被缓存在计算节点的内存中，并供后面重用。

![image-20230131160109393](https://cos.gump.cloud/uPic/image-20230131160109393.png)

0）创建包名： com.atguigu.cache

1）代码实现

```java
package com.atguigu.cache;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;

public class Test01_Cache {
    public static void main(String[] args) throws InterruptedException {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");


        //3.1.业务逻辑
        JavaRDD<String> wordRDD = lineRDD.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String s) throws Exception {
                List<String> stringList = Arrays.asList(s.split(" "));
                return stringList.iterator();
            }
        });

        JavaRDD<Tuple2<String, Integer>> tuple2JavaRDD = wordRDD.map(new Function<String, Tuple2<String, Integer>>() {
            @Override
            public Tuple2<String, Integer> call(String v1) throws Exception {
                System.out.println("*****************");
                return new Tuple2<>(v1, 1);
            }
        });

        //3.5 cache缓存前打印血缘关系
        System.out.println(tuple2JavaRDD.toDebugString());

        //3.4 数据缓存。
//cache底层调用的就是persist方法,缓存级别默认用的是MEMORY_ONLY
        tuple2JavaRDD.cache();

        //3.6 persist方法可以更改存储级别
        // wordToOneRdd.persist(StorageLevel.MEMORY_AND_DISK_2)

        //3.2 触发执行逻辑
        tuple2JavaRDD. collect().forEach(System.out::println);

        //3.5 cache缓存后打印血缘关系
//cache操作会增加血缘关系，不改变原有的血缘关系
        System.out.println(tuple2JavaRDD.toDebugString());
        System.out.println("=====================");

        //3.3 再次触发执行逻辑
        tuple2JavaRDD. collect().forEach(System.out::println);

        Thread.sleep(1000000);

        // 4. 关闭sc
        sc.stop();
    }
}
```

2）源码解析

```scala
mapRdd.cache()
def cache(): this.type = persist()
def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)

object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)
```

注意：默认的存储级别都是仅在内存存储一份。在存储级别的末尾加上 “_2” 表示持久化的数据存为两份。

SER：表示序列化。

![image-20230131160311322](https://cos.gump.cloud/uPic/image-20230131160311322.png)

缓存有可能丢失，或者存储于内存的数据由于内存不足而被删除，{++RDD 的缓存容错机制保证了即使缓存丢失也能保证计算的正确执行++}。通过基于 RDD 的一系列转换，丢失的数据会被重算，由于 RDD 的各个 Partition 是相对独立的，因此只需要计算丢失的部分即可，并不需要重算全部 Partition。

3）自带缓存算子

Spark 会自动对一些 Shuffle 操作的中间数据做持久化操作（比如：reduceByKey）。这样做的目的是为了当一个节点 Shuffle 失败了避免重新计算整个输入。但是，在实际使用的时候，如果想重用数据，仍然建议调用 persist 或 cache。

查看前面 4.7.4 依赖关系代码的 DAG 图

访问 http://localhost:4040/jobs/ 页面，查看第一个和第二个 job 的 DAG 图。

说明：增加缓存后血缘依赖关系仍然有，但是，第二个 job 取的数据是从缓存中取的。


=== "job0"
    ![image-20230131160502186](https://cos.gump.cloud/uPic/image-20230131160502186.png)

=== "job1"
      ![image-20230131160510064](https://cos.gump.cloud/uPic/image-20230131160510064.png)

### 4.8.2 RDD CheckPoint检查点

1）检查点：是通过将 RDD 中间结果{++写入磁盘++}

2）为什么要做检查点？
由于血缘依赖过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果检查点之后有节点出现问题，可以从检查点开始重做血缘，减少了开销

3）检查点存储路径：Checkpoint 的数据通常是存储在 {++HDFS等容错、高可用的文件系统++}

4）检查点数据存储格式为：{++二进制的文件++}

5）{++检查点切断血缘：在 Checkpoint 的过程中，该 RDD 的所有依赖于父 RDD 中的信息将全部被移除++}

6）检查点触发时间：对 RDD 进行 Checkpoint 操作并不会马上被执行，必须执行 Action 操作才能触发。但是检查点为了数据安全，会从血缘关系的最开始执行一遍。

![image-20230131160851492](https://cos.gump.cloud/uPic/image-20230131160851492.png)

7）设置检查点步骤

（1）设置检查点数据存储路径：sc.setCheckpointDir("./checkpoint1")

（2）调用检查点方法：wordToOneRdd.checkpoint()

8）代码实现

```java
package com.atguigu.cache;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

public class Test02_CheckPoint {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        sc.setCheckpointDir("ck");

        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");

        JavaPairRDD<String, Long> tupleRDD = lineRDD.mapToPair(new PairFunction<String, String, Long>() {
            @Override
            public Tuple2<String, Long> call(String s) throws Exception {
                return new Tuple2<String, Long>(s, System.currentTimeMillis());
            }
        });

        // 查看血缘关系
        System.out.println(tupleRDD.toDebugString());

        // 增加检查点避免计算两次
        tupleRDD.cache();

        // 进行检查点
        tupleRDD.checkpoint();

        tupleRDD. collect().forEach(System.out::println);

        System.out.println(tupleRDD.toDebugString());
        // 第二次计算
        tupleRDD. collect().forEach(System.out::println);
        // 第三次计算
        tupleRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

9）执行结果

访问 http://localhost:4040/jobs/ 页面，查看 4 个 job 的 DAG 图。其中第 2 个图是 checkpoint 的 job 运行 DAG 图。第 3、4 张图说明，检查点切断了血缘依赖关系。

=== "stage0"
    <img src="https://cos.gump.cloud/uPic/image-20230131161045693.png" alt="image-20230131161045693" style="zoom:33%;" />

=== "stage1"
    <img src="https://cos.gump.cloud/uPic/image-20230131161102357.png" alt="image-20230131161102357" style="zoom:33%;" />

=== "stage2"
    <img src="https://cos.gump.cloud/uPic/image-20230131161115400.png" alt="image-20230131161115400" style="zoom:33%;" />

=== "stage3"
    <img src="https://cos.gump.cloud/uPic/image-20230131161126863.png" alt="image-20230131161126863" style="zoom:33%;" />

（1）只增加 checkpoint，没有增加 Cache 缓存打印

​	第 1 个 job 执行完，触发了 checkpoint，第 2 个 job 运行 checkpoint，并把数据存储在检查点上。第 3、4 个 job，数据从检查点上直接读取。

```bash
(hadoop,1577960215526)
。。。。。。
(hello,1577960215526)
(hadoop,1577960215609)
。。。。。。
(hello,1577960215609)
(hadoop,1577960215609)
。。。。。。
(hello,1577960215609)
```

（2）增加 checkpoint，也增加 Cache 缓存打印

​	第 1 个 job 执行完，数据就保存到 Cache 里面了，第 2 个 job 运行 checkpoint，直接读取 Cache 里面的数据，并把数据存储在检查点上。第 3、4 个 job，数据从检查点上直接读取。

```bash
(hadoop,1577960642223)
。。。。。。
(hello,1577960642225)
(hadoop,1577960642223)
。。。。。。
(hello,1577960642225)
(hadoop,1577960642223)
。。。。。。
(hello,1577960642225)
```

<figure markdown>
  ![image-20230131161616163](https://cos.gump.cloud/uPic/image-20230131161616163.png)
  <figcaption>CheckPoint检查点+缓存</figcaption>
</figure>
### 4.8.3 缓存和检查点区别

（1）Cache 缓存只是将数据保存起来，不切断血缘依赖。Checkpoint 检查点切断血缘依赖。

（2）Cache 缓存的数据通常存储在磁盘、内存等地方，可靠性低。Checkpoint 的数据通常存储在 HDFS 等容错、高可用的文件系统，可靠性高。

（3）建议对 checkpoint() 的 RDD 使用 Cache 缓存，这样 checkpoint 的 job 只需从 Cache 缓存中读取数据即可，否则需要再从头计算一次 RDD。

（4）如果使用完了缓存，可以通过 unpersist() 方法释放缓存。

### 4.8.4 检查点存储到HDFS集群

如果检查点数据存储到 HDFS 集群，要注意配置访问集群的用户名。否则会报访问权限异常。

```java
package com.atguigu.cache;

import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

public class Test02_CheckPoint2 {
    public static void main(String[] args) {

        // 修改用户名称
        System.setProperty("HADOOP_USER_NAME","atguigu");

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 需要设置路径.需要提前在HDFS集群上创建/checkpoint路径
        sc.setCheckpointDir("hdfs://hadoop102:8020/checkpoint");


        // 3. 编写代码
        JavaRDD<String> lineRDD = sc.textFile("input/2.txt");

        JavaPairRDD<String, Long> tupleRDD = lineRDD.mapToPair(new PairFunction<String, String, Long>() {
            @Override
            public Tuple2<String, Long> call(String s) throws Exception {
                return new Tuple2<String, Long>(s, System.currentTimeMillis());
            }
        });

        // 查看血缘关系
        System.out.println(tupleRDD.toDebugString());

        // 增加检查点避免计算两次
        tupleRDD.cache();

        // 进行检查点
        tupleRDD.checkpoint();

        tupleRDD. collect().forEach(System.out::println);

        System.out.println(tupleRDD.toDebugString());
        // 第二次计算
        tupleRDD. collect().forEach(System.out::println);
        // 第三次计算
        tupleRDD. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

## 4.9 键值对RDD数据分区

Spark 目前支持 {++Hash 分区++}、{++Range 分区++}和{++用户自定义分区++}。Hash 分区为当前的默认分区。分区器直接决定了 RDD 中分区的个数、RDD 中每条数据经过 Shuffle 后进入哪个分区和 Reduce 的个数。

!!! warning "注意"

    （1）只有 Key-Value 类型的 pairRDD 才有分区器，非 Key-Value 类型的 RDD 分区的值是 None
    
    （2）每个 RDD 的分区 ID 范围：0~numPartitions-1，决定这个值是属于那个分区的。

2）获取 RDD 分区

（1）创建包名：com.atguigu.partitioner

（2）代码实现

```java
package com.atguigu.partitioner;

import org.apache.spark.Partitioner;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.Optional;
import org.apache.spark.api.java.function.Function2;
import scala.Tuple2;

import java.util.Arrays;

public class Test01_Partitioner {
    public static void main(String[] args) {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore");

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        JavaPairRDD<String, Integer> tupleRDD = sc.parallelizePairs(Arrays.asList(new Tuple2<>("s", 1), new Tuple2<>("a", 3), new Tuple2<>("d", 2)));

        // 获取分区器
        Optional<Partitioner> partitioner = tupleRDD.partitioner();

        System.out.println(partitioner);

        JavaPairRDD<String, Integer> reduceByKeyRDD = tupleRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        // 获取分区器
        Optional<Partitioner> partitioner1 = reduceByKeyRDD.partitioner();
        System.out.println(partitioner1);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 4.9.1 Hash分区

![image-20230131162228160](https://cos.gump.cloud/uPic/image-20230131162228160.png)

### 4.9.2 Ranger分区

![image-20230131162308986](https://cos.gump.cloud/uPic/image-20230131162308986.png)
