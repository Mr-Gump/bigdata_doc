# 第10章 SparkSQL数据的加载与保存

## 10.1 读取和保存文件

SparkSQL 读取和保存的文件一般为三种，JSON 文件、CSV 文件和列式存储的文件，同时可以通过添加参数，来识别不同的存储和压缩格式。

### 10.1.1 CSV文件

1）代码实现

```java
package com.atguigu.sparksql;

import com.atguigu.sparksql.Bean.User;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.*;

public class Test06_CSV {
    public static void main(String[] args) throws ClassNotFoundException {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        DataFrameReader reader = spark.read();

        // 添加参数  读取csv
        Dataset<Row> userDS = reader
                .option("header", "true")//默认为false 不读取列名
                .option("sep",",") // 默认为, 列的分割
                // 不需要写压缩格式  自适应
                .csv("input/user.csv");

        userDS.show();

        // 转换为user的ds
        // 直接转换类型会报错  csv读取的数据都是string
//        Dataset<User> userDS1 = userDS.as(Encoders.bean(User.class));
        userDS.printSchema();

        Dataset<User> userDS1 = userDS.map(new MapFunction<Row, User>() {
            @Override
            public User call(Row value) throws Exception {
                return new User(Long.valueOf(value.getString(0)), value.getString(1));
            }
        }, Encoders.bean(User.class));

        userDS1.show();

        // 写出为csv文件
        DataFrameWriter<User> writer = userDS1.write();

        writer.option("header",";")
                .option("header","true")
//                .option("compression","gzip")// 压缩格式
                // 写出模式
                // append 追加
                // Ignore 忽略本次写出
                // Overwrite 覆盖写
                // ErrorIfExists 如果存在报错
                .mode(SaveMode.Append)
                .csv("output");

        //4. 关闭sparkSession
        spark.close();
    }
}
```

### 10.1.2 JSON文件

```java
package com.atguigu.sparksql;

import com.atguigu.sparksql.Bean.User;
import org.apache.spark.SparkConf;
import org.apache.spark.sql.*;

public class Test07_JSON {
    public static void main(String[] args) {
        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        Dataset<Row> json = spark.read().json("input/user.json");

        // json数据可以读取数据的数据类型
        Dataset<User> userDS = json.as(Encoders.bean(User.class));

        userDS.show();
        
        // 读取别的类型的数据也能写出为json
        DataFrameWriter<User> writer = userDS.write();

        writer.json("output1");

        //4. 关闭sparkSession
        spark.close();
    }
}
3.1.3 Parquet文件
列式存储的数据自带列分割。
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

public class Test08_Parquet {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        Dataset<Row> json = spark.read().json("input/user.json");

        // 写出默认使用snappy压缩
//        json.write().parquet("output");

        // 读取parquet 自带解析  能够识别列名
        Dataset<Row> parquet = spark.read().parquet("output");

        parquet.printSchema();

        //4. 关闭sparkSession
        spark.close();
    }
}
```

## 10.2 与MySQL交互

1）导入依赖

```xml title="pom.xml"
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.27</version>
</dependency>
```

2）从 MySQL 读数据

```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;

import java.util.Properties;

public class Test09_Table {
    public static void main(String[] args) {

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder().config(conf).getOrCreate();

        //3. 编写代码
        Dataset<Row> json = spark.read().json("input/user.json");

        // 添加参数
        Properties properties = new Properties();
        properties.setProperty("user","root");
        properties.setProperty("password","000000");

//        json.write()
//                // 写出模式针对于表格追加覆盖
//                .mode(SaveMode.Append)
//                .jdbc("jdbc:mysql://hadoop102:3306","gmall.testInfo",properties);

        Dataset<Row> jdbc = spark.read().jdbc("jdbc:mysql://hadoop102:3306", "gmall.testInfo", properties);

        jdbc.show();


        //4. 关闭sparkSession
        spark.close();
    }
}
```

## 10.3 与Hive交互

SparkSQL 可以采用内嵌 Hive（ Spark 开箱即用的 Hive），也可以采用外部 Hive。{==企业开发中，通常采用外部 Hive==}。

### 10.3.1 Linux中的交互

1）添加 MySQL 连接驱动到 spark-yarn 的 jars 目录

```bash
cp /opt/software/mysql-connector-java-5.1.27-bin.jar /opt/module/spark-yarn/jars
```

2）添加 hive-site.xml 文件到 spark-yarn 的 conf 目录

```bash
cp /opt/module/hive/conf/hive-site.xml /opt/module/spark-yarn/conf
```

3）启动 spark-sql 的客户端即可

<div class="termy">
```console
$ bin/spark-sql --master yarn
spark-sql (default)> show tables;
```
</div>

### 10.3.2 IDEA中的交互

1）添加依赖

```pom title="pom.xml"
<dependencies>
    <dependency>
       <groupId>org.apache.spark</groupId>
       <artifactId>spark-sql_2.12</artifactId>
       <version>3.1.3</version>
    </dependency>

    <dependency>
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId>
       <version>5.1.27</version>
    </dependency>

    <dependency>
       <groupId>org.apache.spark</groupId>
       <artifactId>spark-hive_2.12</artifactId>
       <version>3.1.3</version>
    </dependency>

    <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <version>1.18.22</version>
    </dependency>
</dependencies>
```

2）拷贝 {==hive-site.xml==} 到 resources 目录（如果需要操作 Hadoop，需要拷贝 hdfs-site.xml、core-site.xml、yarn-site.xml）

3）代码实现

```java
package com.atguigu.sparksql;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.SparkSession;

public class Test10_Hive {
    public static void main(String[] args) {

        System.setProperty("HADOOP_USER_NAME","atguigu");

        //1. 创建配置对象
        SparkConf conf = new SparkConf().setAppName("sparksql").setMaster("local[*]");

        //2. 获取sparkSession
        SparkSession spark = SparkSession.builder()
                .enableHiveSupport()// 添加hive支持
                .config(conf).getOrCreate();

        //3. 编写代码
        spark.sql("show tables").show();

        spark.sql("create table user_info(name String,age bigint)");
        spark.sql("insert into table user_info values('zhangsan',10)");
        spark.sql("select * from user_info").show();

        //4. 关闭sparkSession
        spark.close();
    }
}
```

