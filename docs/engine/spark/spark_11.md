# 第11章 SparkSQL项目实战

## 11.1 准备数据

我们这次 SparkSQL 操作所有的数据均来自 Hive，首先在 Hive 中创建表，并导入数据。一共有 3 张表：1 张用户行为表，1 张城市表，1 张产品表。

1）将 city_info.txt、product_info.txt、user_visit_action.txt 上传到 /opt/module/data

```bash
mkdir data
```

2）将创建对应的三张表

```sql
CREATE TABLE `user_visit_action`(
  `date` string,
  `user_id` bigint,
  `session_id` string,
  `page_id` bigint,
  `action_time` string,
  `search_keyword` string,
  `click_category_id` bigint,
  `click_product_id` bigint, --点击商品id，没有商品用-1表示。
  `order_category_ids` string,
  `order_product_ids` string,
  `pay_category_ids` string,
  `pay_product_ids` string,
  `city_id` bigint --城市id
)
row format delimited fields terminated by '\t';


CREATE TABLE `city_info`(
  `city_id` bigint, --城市id
  `city_name` string, --城市名称
  `area` string --区域名称
)
row format delimited fields terminated by '\t';


CREATE TABLE `product_info`(
  `product_id` bigint, -- 商品id
  `product_name` string, --商品名称
  `extend_info` string
)
row format delimited fields terminated by '\t';
```

3）并加载数据

```sql
load data local inpath '/opt/module/data/user_visit_action.txt' into table user_visit_action;
load data local inpath '/opt/module/data/product_info.txt' into table product_info;
load data local inpath '/opt/module/data/city_info.txt' into table city_info;
```

4）测试一下三张表数据是否正常

```sql
select * from user_visit_action limit 5;
select * from product_info limit 5;
select * from city_info limit 5;
```

## 11.2 需求：各区域热门商品Top3

### 11.2.1 需求简介

这里的热门商品是从{==点击量==}的维度来看的，计算各个区域前三大热门商品，并备注上每个商品在主要城市中的分布比例，超过两个城市用其他显示。

例如：

| 地区 | 商品名称 | 点击次数 | 城市备注                        |
| ---- | -------- | -------- | ------------------------------- |
| 华北 | 商品A    | 100000   | 北京21.2%，天津13.2%，其他65.6% |
| 华北 | 商品P    | 80200    | 北京63.0%，太原10%，其他27.0%   |
| 华北 | 商品M    | 40000    | 北京63.0%，太原10%，其他27.0%   |
| 东北 | 商品J    | 92000    | 大连28%，辽宁17.0%，其他 55.0%  |

### 11.2.2 思路分析

![image-20230206185928473](https://cos.gump.cloud/uPic/image-20230206185928473.png)

使用 SparkSQL 来完成复杂的需求，可以使用 UDF 或 UDAF。

（1）查询出来所有的点击记录，并与 city_info 表连接，得到每个城市所在的地区，与 Product_info 表连接得到商品名称。
（2）按照地区和商品名称分组，统计出每个商品在每个地区的总点击次数。
（3）每个地区内按照点击次数降序排列。
（4）只取前三名，并把结果保存在数据库中。
（5）城市备注需要自定义 UDAF 函数。

### 11.2.3 代码实现

```java
package com.atguigu.sparksql.demo;

import lombok.Data;
import org.apache.spark.SparkConf;
import org.apache.spark.sql.*;
import org.apache.spark.sql.expressions.Aggregator;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.TreeMap;
import java.util.function.BiConsumer;
import static org.apache.spark.sql.functions.udaf;

public class Test01_Top3 {
    public static void main(String[] args) {

        // 1. 创建sparkConf配置对象
        SparkConf conf = new SparkConf().setAppName("sql").setMaster("local[*]");

        // 2. 创建sparkSession连接对象
        SparkSession spark = SparkSession.builder().enableHiveSupport().config(conf).getOrCreate();

        // 3. 编写代码
        // 将3个表格数据join在一起
        Dataset<Row> t1DS = spark.sql("select \n" +
                "\tc.area,\n" +
                "\tc.city_name,\n" +
                "\tp.product_name\n" +
                "from\n" +
                "\tuser_visit_action u\n" +
                "join\n" +
                "\tcity_info c\n" +
                "on\n" +
                "\tu.city_id=c.city_id\n" +
                "join\n" +
                "\tproduct_info p\n" +
                "on\n" +
                "\tu.click_product_id=p.product_id");

        t1DS.createOrReplaceTempView("t1");

        spark.udf().register("cityMark",udaf(new CityMark(),Encoders.STRING()));

        // 将区域内的产品点击次数统计出来
        Dataset<Row> t2ds = spark.sql("select \n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tcityMark(city_name) mark,\n" +
                "\tcount(*) counts\n" +
                "from\t\n" +
                "\tt1\n" +
                "group by\n" +
                "\tarea,product_name");

//        t2ds.show(false);
        t2ds.createOrReplaceTempView("t2");

        // 对区域内产品点击的次数进行排序  找出区域内的top3
        spark.sql("select\n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tmark,\n" +
                "\trank() over (partition by area order by counts desc) rk\n" +
                "from \n" +
                "\tt2").createOrReplaceTempView("t3");

        // 使用过滤  取出区域内的top3
        spark.sql("select\n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tmark \n" +
                "from\n" +
                "\tt3\n" +
                "where \n" +
                "\trk < 4").show(50,false);

        // 4. 关闭sparkSession
        spark.close();
    }

    @Data
    public static class Buffer implements Serializable {
        private Long totalCount;
        private HashMap<String,Long> map;

        public Buffer() {
        }

        public Buffer(Long totalCount, HashMap<String, Long> map) {
            this.totalCount = totalCount;
            this.map = map;
        }
    }

    public static class CityMark extends Aggregator<String,Buffer,String>{

        @Override
        public Buffer zero() {
            return new Buffer(0L,new HashMap<String,Long>());
        }

        /**
         * 分区内的预聚合
         * @param b  map(城市,sum)
         * @param a  当前行表示的城市
         * @return
         */
        @Override
        public Buffer reduce(Buffer b, String a) {
            HashMap<String, Long> hashMap = b.getMap();
            // 如果map中已经有当前城市  次数+1
            // 如果map中没有当前城市    0+1
            hashMap.put(a, hashMap.getOrDefault(a,0L) + 1);

            b.setTotalCount(b.getTotalCount() + 1L);
            return b;
        }

        /**
         * 合并多个分区间的数据
         * @param b1    (北京,100),(上海,200)
         * @param b2    (天津,100),(上海,200)
         * @return
         */
        @Override
        public Buffer merge(Buffer b1, Buffer b2) {
            b1.setTotalCount( b1.getTotalCount() + b2.getTotalCount());

            HashMap<String, Long> map1 = b1.getMap();
            HashMap<String, Long> map2 = b2.getMap();
            // 将map2中的数据放入合并到map1
            map2.forEach(new BiConsumer<String, Long>() {
                @Override
                public void accept(String s, Long aLong) {
                    map1.put(s,aLong + map1.getOrDefault(s,0L));
                }
            });

            return b1;
        }

        /**
         * map => {(上海,200),(北京,100),(天津,300)}
         * @param reduction
         * @return
         */
        @Override
        public String finish(Buffer reduction) {
            Long totalCount = reduction.getTotalCount();
            HashMap<String, Long> map = reduction.getMap();
            // 需要对map中的value次数进行排序
            TreeMap<Long, String> treeMap = new TreeMap<>();

            // 将map中的数据放入到treeMap中 进行排序
            map.forEach(new BiConsumer<String, Long>() {
                @Override
                public void accept(String s, Long aLong) {
                    if (treeMap.containsKey(aLong)){
                        // 如果已经有当前值
                        treeMap.put(aLong,treeMap.get(aLong) + "_" + s );
                    }else {
                        // 没有当前值
                        treeMap.put(aLong,s );
                    }
                }
            });

            ArrayList<String> resultMark = new ArrayList<>();

            Double sum = 0.0;

            // 当前没有更多的城市数据  或者  已经找到两个城市数据了  停止循环
            while(!(treeMap.size() == 0) && resultMark.size() < 2){
                String cities = treeMap.lastEntry().getValue();
                Long counts = treeMap.lastEntry().getKey();
                String[] strings = cities.split("_");
                for (String city : strings) {
                    double rate = counts.doubleValue() * 100 / totalCount;
                    sum += rate;
                    resultMark.add(city + String.format("%.2f",rate) + "%");
                }
                // 添加完成之后删除当前key
                treeMap.remove(counts);
            }

            // 拼接其他城市
            if (treeMap.size() > 0 ){
                resultMark.add("其他" + String.format("%.2f",100 - sum) + "%");
            }

            StringBuilder cityMark = new StringBuilder();
            for (String s : resultMark) {
                cityMark.append(s).append(",");
            }

            return cityMark.substring(0, cityMark.length() - 1);
        }

        @Override
        public Encoder<Buffer> bufferEncoder() {
            return Encoders.javaSerialization(Buffer.class);
        }

        @Override
        public Encoder<String> outputEncoder() {
            return Encoders.STRING();
        }
    }
}
```

### 11.2.4 集群实现

实际开发是在集群上进行的，模拟真实的开发环境，用户行为日志为 json 格式，存放在 HDFS 上面，最终的结果作为表格存放在 Hive 中。

1）代码编写

```java
package com.atguigu.sparksql;

import lombok.Data;
import org.apache.spark.SparkConf;
import org.apache.spark.sql.*;
import org.apache.spark.sql.expressions.Aggregator;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.TreeMap;
import java.util.function.BiConsumer;
import static org.apache.spark.sql.functions.udaf;


public class Test02_Top3 {
    public static void main(String[] args) {
        System.setProperty("HADOOP_USER_NAME", "atguigu");


        // 1. 创建sparkConf配置对象
        SparkConf conf = new SparkConf().setAppName("sql");
                // 上传集群使用注释掉
//                .setMaster("local[*]");

        // 2. 创建sparkSession连接对象
        SparkSession spark = SparkSession.builder().enableHiveSupport().config(conf).getOrCreate();

        // 3. 编写代码

        // 读取hdfs上面的json数据
        // 测试先使用本地
        // 集群使用main方法参数作为写入路径
        spark.read().json(args[0]).createOrReplaceTempView("user_visit_action");

        // 将3个表格数据join在一起
        Dataset<Row> t1DS = spark.sql("select \n" +
                "\tc.area,\n" +
                "\tc.city_name,\n" +
                "\tp.product_name\n" +
                "from\n" +
                "\tuser_visit_action u\n" +
                "join\n" +
                "\tcity_info c\n" +
                "on\n" +
                "\tu.city_id=c.city_id\n" +
                "join\n" +
                "\tproduct_info p\n" +
                "on\n" +
                "\tu.click_product_id=p.product_id");

        t1DS.createOrReplaceTempView("t1");

        spark.udf().register("cityMark",udaf(new CityMark(),Encoders.STRING()));


        // 将区域内的产品点击次数统计出来
        Dataset<Row> t2ds = spark.sql("select \n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tcityMark(city_name) mark,\n" +
                "\tcount(*) counts\n" +
                "from\t\n" +
                "\tt1\n" +
                "group by\n" +
                "\tarea,product_name");

//        t2ds.show(false);
        t2ds.createOrReplaceTempView("t2");

        // 对区域内产品点击的次数进行排序  找出区域内的top3
        spark.sql("select\n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tmark,\n" +
                "\trank() over (partition by area order by counts desc) rk\n" +
                "from \n" +
                "\tt2").createOrReplaceTempView("t3");

        // 使用过滤  取出区域内的top3
        // 同时结果写出到表格中
        spark.sql("drop table if exists result");
        spark.sql("create table result as \n" +
                "select \n" +
                "\tarea,\n" +
                "\tproduct_name,\n" +
                "\tmark\n" +
                "from\n" +
                "\tt3\n" +
                "where\n" +
                "\trk < 4");


        // 4. 关闭sparkSession
        spark.close();
    }

    @Data
    public static class Buffer implements Serializable {
        private Long totalCount;
        private HashMap<String,Long> map;

        public Buffer() {
        }

        public Buffer(Long totalCount, HashMap<String, Long> map) {
            this.totalCount = totalCount;
            this.map = map;
        }
    }

    public static class CityMark extends Aggregator<String,Buffer,String>{

        @Override
        public Buffer zero() {
            return new Buffer(0L,new HashMap<String,Long>());
        }

        /**
         * 分区内的预聚合
         * @param b  map(城市,sum)
         * @param a  当前行表示的城市
         * @return
         */
        @Override
        public Buffer reduce(Buffer b, String a) {
            HashMap<String, Long> hashMap = b.getMap();
            // 如果map中已经有当前城市  次数+1
            // 如果map中没有当前城市    0+1
            hashMap.put(a, hashMap.getOrDefault(a,0L) + 1);

            b.setTotalCount(b.getTotalCount() + 1L);
            return b;
        }

        /**
         * 合并多个分区间的数据
         * @param b1    (北京,100),(上海,200)
         * @param b2    (天津,100),(上海,200)
         * @return
         */
        @Override
        public Buffer merge(Buffer b1, Buffer b2) {
            b1.setTotalCount( b1.getTotalCount() + b2.getTotalCount());

            HashMap<String, Long> map1 = b1.getMap();
            HashMap<String, Long> map2 = b2.getMap();
            // 将map2中的数据放入合并到map1
            map2.forEach(new BiConsumer<String, Long>() {
                @Override
                public void accept(String s, Long aLong) {
                    map1.put(s,aLong + map1.getOrDefault(s,0L));
                }
            });

            return b1;
        }

        /**
         * map => {(上海,200),(北京,100),(天津,300)}
         * @param reduction
         * @return
         */
        @Override
        public String finish(Buffer reduction) {
            Long totalCount = reduction.getTotalCount();
            HashMap<String, Long> map = reduction.getMap();
            // 需要对map中的value次数进行排序
            TreeMap<Long, String> treeMap = new TreeMap<>();

            // 将map中的数据放入到treeMap中 进行排序
            map.forEach(new BiConsumer<String, Long>() {
                @Override
                public void accept(String s, Long aLong) {
                    if (treeMap.containsKey(aLong)){
                        // 如果已经有当前值
                        treeMap.put(aLong,treeMap.get(aLong) + "_" + s );
                    }else {
                        // 没有当前值
                        treeMap.put(aLong,s );
                    }
                }
            });

            ArrayList<String> resultMark = new ArrayList<>();

            Double sum = 0.0;

            // 当前没有更多的城市数据  或者  已经找到两个城市数据了  停止循环
            while(!(treeMap.size() == 0) && resultMark.size() < 2){
                String cities = treeMap.lastEntry().getValue();
                Long counts = treeMap.lastEntry().getKey();
                String[] strings = cities.split("_");
                for (String city : strings) {
                    double rate = counts.doubleValue() * 100 / totalCount;
                    sum += rate;
                    resultMark.add(city + String.format("%.2f",rate) + "%");
                }
                // 添加完成之后删除当前key
                treeMap.remove(counts);
            }

            // 拼接其他城市
            if (treeMap.size() > 0 ){
                resultMark.add("其他" + String.format("%.2f",100 - sum) + "%");
            }

            StringBuilder cityMark = new StringBuilder();
            for (String s : resultMark) {
                cityMark.append(s).append(",");
            }

            return cityMark.substring(0, cityMark.length() - 1);
        }

        @Override
        public Encoder<Buffer> bufferEncoder() {
            return Encoders.javaSerialization(Buffer.class);
        }

        @Override
        public Encoder<String> outputEncoder() {
            return Encoders.STRING();
        }
    }
}
```

2）修改依赖打包上传

```xml title="pom.xml"
<dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.12</artifactId>
            <version>3.1.3</version>
            <scope>provided</scope>
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
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.22</version>
        </dependency>
</dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <excludes>
                                    <exclude>com.google.code.findbugs:jsr305</exclude>
                                    <exclude>org.slf4j:*</exclude>
                                    <exclude>log4j:*</exclude>
                                    <exclude>org.apache.hadoop:*</exclude>
                                </excludes>
                            </artifactSet>
                            <filters>
                                <filter>
                                    <!-- Do not copy the signatures in the META-INF folder.Otherwise, this might cause SecurityExceptions when using the JAR. -->
                                    <!-- 打包时不复制META-INF下的签名文件，避免报非法签名文件的SecurityExceptions异常-->
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>

                            <transformers combine.children="append">
                                <!-- The service transformer is needed to merge META-INF/services files -->
                                <!-- connector和format依赖的工厂类打包时会相互覆盖，需要使用ServicesResourceTransformer解决-->
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

3）上传数据到 HDFS 

```bash
hadoop fs -mkdir /input
hadoop fs -put user_visti_action /input
```

4）之后使用 Spark 的任务提交命令

```bash
bin/spark-submit --master yarn --class com.atguigu.sparksql.Test02_Top3 ./sparkSQL-1.0-SNAPSHOT.jar /input/user_visit_action.json
```

5）查看结果
<div class="termy">
```console
$ select * from result;

area	product_name	mark
华东	商品_86	上海16.44%,杭州15.90%,无锡15.90%,其他51.75%
华东	商品_47	杭州15.85%,青岛15.57%,其他68.58%
华东	商品_75	上海17.49%,无锡15.57%,其他66.94%
西北	商品_15	西安54.31%,银川45.69%
西北	商品_2	银川53.51%,西安46.49%
西北	商品_22	西安54.87%,银川45.13%
华南	商品_23	厦门29.02%,福州24.55%,深圳24.55%,其他21.88%
华南	商品_65	深圳27.93%,厦门26.58%,其他45.50%
华南	商品_50	福州27.36%,深圳25.94%,其他46.70%
华北	商品_42	保定25.00%,郑州25.00%,其他50.00%
华北	商品_99	北京24.24%,郑州23.48%,其他52.27%
华北	商品_19	郑州23.46%,保定20.38%,其他56.15%
东北	商品_41	哈尔滨35.50%,大连34.91%,其他29.59%
东北	商品_91	哈尔滨35.76%,大连32.73%,其他31.52%
东北	商品_58	沈阳37.74%,大连32.08%,其他30.19%
东北	商品_93	哈尔滨38.36%,大连37.11%,其他24.53%
华中	商品_62	武汉51.28%,长沙48.72%
华中	商品_4	长沙53.10%,武汉46.90%
华中	商品_57	武汉54.95%,长沙45.05%
华中	商品_29	武汉50.45%,长沙49.55%
西南	商品_1	成都35.80%,贵阳35.80%,其他28.41%
西南	商品_44	贵阳37.28%,成都34.32%,其他28.40%
西南	商品_60	重庆39.88%,成都30.67%,其他29.45%
```
</div>
