# 第9章 Spark SQL编程

## 9.1 SparkSession新的起始点

在老的版本中，SparkSQL 提供两种 SQL 查询起始点：

- SQLContext，用于 Spark 自己提供的 SQL 查询
- HiveContext，用于连接 Hive 的查询

SparkSession 是 Spark {==最新==}的 SQL 查询起始点，实质上是 SQLContext 和 HiveContext 的组合，所以在 SQLContext 和 HiveContext 上可用的 API 在 SparkSession 上同样是可以使用的。

SparkSession 内部封装了 SparkContext，所以计算实际上是由 SparkContext 完成的。当我们使用 spark-shell 的时候，Spark 框架会自动的创建一个名称叫做 Spark 的 SparkSession，就像我们以前可以自动获取到一个 sc 来表示 SparkContext。

<div class="termy">
```console
$  bin/spark-shell

20/09/12 11:16:35 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://hadoop102:4040
Spark context available as 'sc' (master = local[*], app id = local-1599880621394).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.1.3
      /_/
         
Using Scala version 2.12.10 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_212)
Type in expressions to have them evaluated.
Type :help for more information.
```
</div>

## 9.2 常用方式
### 9.2.1 方法调用

1）创建一个 maven 工程 SparkSQL

2）创建包名为 com.atguigu.sparksql

3）输入文件夹准备：在新建的 SparkSQL 项目名称上 右键 =》新建 input 文件夹 =》在 input 文件夹上右键 =》 新建 user.json。并输入如下内容：
```json title="user.json"
{"age":20,"name":"qiaofeng"}
{"age":19,"name":"xuzhu"}
{"age":18,"name":"duanyu"}
{"age":22,"name":"qiaofeng"}
{"age":11,"name":"xuzhu"}
{"age":12,"name":"duanyu"}
```

5）在 pom.xml 文件中添加 spark-sql 的依赖
```xml title="pom.xml"
<dependencies>
    <dependency>
       <groupId>org.apache.spark</groupId>
       <artifactId>spark-sql_2.12</artifactId>
       <version>3.1.3</version>
    </dependency>

    <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <version>1.18.22</version>
    </dependency>
</dependencies>
```
6）代码实现

添加 javaBean 的 User
```java
package com.atguigu.sparksql.Bean;

import lombok.Data;

import java.io.Serializable;

@Data
public class User implements Serializable {
    public Long age;
    public String name;

    public User() {
    }

    public User(Long age, String name) {
        this.age = age;
        this.name = name;
    }
}
```

代码编写
```java
package com.atguigu.sparksql;

import com.atguigu.sparksql.Bean.User;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.api.java.function.ReduceFunction;
import org.apache.spark.sql.*;
import scala.Tuple2;

public class Test01_Method {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        // 按照行读取
        Dataset<Row> lineDS = spark.read().json("input/user.json");

        // 转换为类和对象
        Dataset<User> userDS = lineDS.as(Encoders.bean(User.class));

//        userDS.show();

        // 使用方法操作
        // 函数式的方法
        Dataset<User> userDataset = lineDS.map(new MapFunction<Row, User>() {
            @Override
            public User call(Row value) throws Exception {
                return new User(value.getLong(0), value.getString(1));
            }
        },
                // 使用kryo在底层会有部分算子无法使用
                Encoders.bean(User.class));

        // 常规方法
        Dataset<User> sortDS = userDataset.sort(new Column("age"));
        sortDS.show();

        // 区分
        RelationalGroupedDataset groupByDS = userDataset.groupBy("name");

        // 后续方法不同
        Dataset<Row> count = groupByDS.count();

        // 推荐使用函数式的方法  使用更灵活
        KeyValueGroupedDataset<String, User> groupedDataset = userDataset.groupByKey(new MapFunction<User, String>() {
            @Override
            public String call(User value) throws Exception {

                return value.name;
            }
        }, Encoders.STRING());

        // 聚合算子都是从groupByKey开始
        // 推荐使用reduceGroup
        Dataset<Tuple2<String, User>> result = groupedDataset.reduceGroups(new ReduceFunction<User>() {
            @Override
            public User call(User v1, User v2) throws Exception {
                // 取用户的大年龄
                return new User(Math.max(v1.age, v2.age), v1.name);
            }
        });

        result.show();

        //4. 关闭sparkSession
        spark.close();
    }
}
```

在 sparkSql 中 DS 直接支持的转换算子有：
- map（底层已经优化为 mapPartition）
- mapPartition
- flatMap
- groupByKey（聚合算子全部由 groupByKey 开始）
- filter
- distinct
- coalesce
- repartition
- sort 和  orderBy（{==不是函数式的算子，不过不影响使用==}）

### 9.2.2 SQL使用方式
```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

public class Test02_SQL {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        Dataset<Row> lineDS = spark.read().json("input/user.json");

        // 创建视图 => 转换为表格 填写表名
        // 临时视图的生命周期和当前的sparkSession绑定
        // orReplace表示覆盖之前相同名称的视图
        lineDS.createOrReplaceTempView("t1");

        // 支持所有的hive sql语法,并且会使用spark的又花钱
        Dataset<Row> result = spark.sql("select * from t1 where age > 18");

        result.show();

        //4. 关闭sparkSession
        spark.close();
    }
}}
```
### 9.2.3 DSL特殊语法（扩展）
```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

import static org.apache.spark.sql.functions.col;

public class Test03_DSL {
    public static void main(String[] args) {
        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        // 导入特殊的依赖 import static org.apache.spark.sql.functions.col;
        Dataset<Row> lineRDD = spark.read().json("input/user.json");

        Dataset<Row> result = lineRDD.select(col("name").as("newName"),col("age").plus(1).as("newAge"))
                .filter(col("age").gt(18));

        result.show();

        //4. 关闭sparkSession
        spark.close();
    }
}
```
## 9.3 SQL语法的用户自定义函数
### 9.3.1 UDF
1）UDF：一行进入，一行出

2）代码实现
```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.api.java.UDF1;
import org.apache.spark.sql.expressions.UserDefinedFunction;
import org.apache.spark.sql.types.DataTypes;

import static org.apache.spark.sql.functions.udf;

public class Test04_UDF {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        Dataset<Row> lineRDD = spark.read().json("input/user.json");

        lineRDD.createOrReplaceTempView("user");

        // 定义一个函数
        // 需要首先导入依赖import static org.apache.spark.sql.functions.udf;
        UserDefinedFunction addName = udf(new UDF1<String, String>() {
            @Override
            public String call(String s) throws Exception {
                return s + " 大侠";
            }
        }, DataTypes.StringType);

        spark.udf().register("addName",addName);

        spark.sql("select addName(name) newName from user")
                .show();

        // lambda表达式写法
        spark.udf().register("addName1",(UDF1<String,String>) name -> name + " 大侠",DataTypes.StringType);

        //4. 关闭sparkSession
        spark.close();
    }
}
```

### 9.3.2 UDAF
1）UDAF：输入多行，返回一行。通常和 groupBy 一起使用，如果直接使用 UDAF 函数，默认将所有的数据合并在一起。

2）Spark3.x 推荐使用 extends Aggregator 自定义 UDAF，属于强类型的 Dataset 方式。

3）Spark2.x 使用 extends UserDefinedAggregateFunction，属于弱类型的 DataFrame

4）案例实操

需求：实现求平均年龄，自定义 UDAF，MyAvg(age)

自定义聚合函数实现-强类型
```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.expressions.Aggregator;

import java.io.Serializable;

import static org.apache.spark.sql.functions.udaf;

public class Test05_UDAF {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        spark.read().json("input/user.json").createOrReplaceTempView("user");

        // 注册需要导入依赖 import static org.apache.spark.sql.functions.udaf;
        spark.udf().register("avgAge",udaf(new MyAvg(),Encoders.LONG()));

        spark.sql("select avgAge(age) newAge from user").show();

        //4. 关闭sparkSession
        spark.close();
    }

    public static class Buffer implements Serializable {
        private Long sum;
        private Long count;

        public Buffer() {
        }

        public Buffer(Long sum, Long count) {
            this.sum = sum;
            this.count = count;
        }

        public Long getSum() {
            return sum;
        }

        public void setSum(Long sum) {
            this.sum = sum;
        }

        public Long getCount() {
            return count;
        }

        public void setCount(Long count) {
            this.count = count;
        }
    }

    public static class MyAvg extends Aggregator<Long,Buffer,Double>{

        @Override
        public Buffer zero() {
            return new Buffer(0L,0L);
        }

        @Override
        public Buffer reduce(Buffer b, Long a) {
            b.setSum(b.getSum() + a);
            b.setCount(b.getCount() + 1);
            return b;
        }

        @Override
        public Buffer merge(Buffer b1, Buffer b2) {

            b1.setSum(b1.getSum() + b2.getSum());
            b1.setCount(b1.getCount() + b2.getCount());

            return b1;
        }

        @Override
        public Double finish(Buffer reduction) {
            return reduction.getSum().doubleValue() / reduction.getCount();
        }

        @Override
        public Encoder<Buffer> bufferEncoder() {
            // 可以用kryo进行优化
            return Encoders.kryo(Buffer.class);
        }

        @Override
        public Encoder<Double> outputEncoder() {
            return Encoders.DOUBLE();
        }
    }
}
```

### 9.3.3 UDTF（没有）
输入一行，返回多行（ Hive ）。

SparkSQL 中没有 UDTF，需要使用算子类型的 flatMap 先完成拆分。

