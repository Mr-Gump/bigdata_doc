# 第5章 SparkCore实战

## 7.1 数据准备

1）数据格式

![image-20230131162555183](https://cos.gump.cloud/uPic/image-20230131162555183.png)

2）数据详细字段说明

| 编号 |      字段名称      | 字段类型 |          字段含义          |
| :--: | :----------------: | :------: | :------------------------: |
|  1   |        date        |  String  |     用户点击行为的日期     |
|  2   |      user_id       |   Long   |          用户的ID          |
|  3   |     session_id     |  String  |        Session的ID         |
|  4   |      page_id       |   Long   |        某个页面的ID        |
|  5   |    action_time     |  String  |        动作的时间点        |
|  6   |   search_keyword   |  String  |      用户搜索的关键词      |
|  7   | click_category_id  |   Long   |   点击某一个商品品类的ID   |
|  8   |  click_product_id  |   Long   |       某一个商品的ID       |
|  9   | order_category_ids |  String  | 一次订单中所有品类的ID集合 |
|  10  | order_product_ids  |  String  | 一次订单中所有商品的ID集合 |
|  11  |  pay_category_ids  |  String  | 一次支付中所有品类的ID集合 |
|  12  |  pay_product_ids   |  String  | 一次支付中所有商品的ID集合 |
|  13  |      city_id       |   Long   |          城市 id           |

## 7.2 需求：Top10热门品类

![image-20230131162652816](https://cos.gump.cloud/uPic/image-20230131162652816.png)

需求说明：品类是指产品的分类，大型电商网站品类分多级，咱们的项目中品类只有一级，不同的公司可能对热门的定义不一样。我们按照每个品类的点击、下单、支付的量（次数）来统计热门品类。

鞋			点击数 下单数  支付数
衣服		点击数 下单数  支付数
电脑		点击数 下单数  支付数
例如，{==综合排名 = 点击数*20% + 下单数*30% + 支付数*50%==}
为了更好的泛用性，当前案例按照点击次数进行排序，如果点击相同，按照下单数，如果下单还是相同，按照支付数。

### 7.2.1 需求分析（方案一）类对象

采用样例类的方式实现。

### 7.2.2 需求实现（方案一）

1）添加 lombok 的插件

![image-20230131162810910](https://cos.gump.cloud/uPic/image-20230131162810910.png)

2）添加依赖 lombok 可以省略 getset 代码

```xml title="pom.xml"
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.22</version>
</dependency>
```

3）创建两个存放数据的类

```java
package com.atguigu.spark.bean;

import lombok.Data;

import java.io.Serializable;

@Data
public class UserVisitAction implements Serializable {
    private String date;
    private String user_id;
    private String session_id;
    private String page_id;
    private String action_time;
    private String search_keyword;
    private String click_category_id;
    private String click_product_id;
    private String order_category_ids;
    private String order_product_ids;
    private String pay_category_ids;
    private String pay_product_ids;
    private String city_id;

    public UserVisitAction() {
    }

    public UserVisitAction(String date, String user_id, String session_id, String page_id, String action_time, String search_keyword, String click_category_id, String click_product_id, String order_category_ids, String order_product_ids, String pay_category_ids, String pay_product_ids, String city_id) {
        this.date = date;
        this.user_id = user_id;
        this.session_id = session_id;
        this.page_id = page_id;
        this.action_time = action_time;
        this.search_keyword = search_keyword;
        this.click_category_id = click_category_id;
        this.click_product_id = click_product_id;
        this.order_category_ids = order_category_ids;
        this.order_product_ids = order_product_ids;
        this.pay_category_ids = pay_category_ids;
        this.pay_product_ids = pay_product_ids;
        this.city_id = city_id;
    }

}
```

```java
package com.atguigu.spark.bean;

import lombok.Data;

import java.io.Serializable;

@Data
public class CategoryCountInfo implements Serializable, Comparable<CategoryCountInfo> {
    private String categoryId;
    private Long clickCount;
    private Long orderCount;
    private Long payCount;

    public CategoryCountInfo() {
    }

    public CategoryCountInfo(String categoryId, Long clickCount, Long orderCount, Long payCount) {
        this.categoryId = categoryId;
        this.clickCount = clickCount;
        this.orderCount = orderCount;
        this.payCount = payCount;
    }


    @Override
    public int compareTo(CategoryCountInfo o) {
        // 小于返回-1,等于返回0,大于返回1
        if (this.getClickCount().equals(o.getClickCount())) {
            if (this.getOrderCount().equals(o.getOrderCount())) {
                if (this.getPayCount().equals(o.getPayCount())) {
                    return 0;
                } else {
                    return this.getPayCount() < o.getPayCount() ? -1 : 1;
                }
            } else {
                return this.getOrderCount() < o.getOrderCount() ? -1 : 1;
            }
        } else {
            return this.getClickCount() < o.getClickCount() ? -1 : 1;
        }

    }
}
```

4）核心业务代码实现

```java
package com.atguigu.spark.demo;

import com.atguigu.spark.bean.CategoryCountInfo;
import com.atguigu.spark.bean.UserVisitAction;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function;
import scala.Tuple2;
import scala.Tuple3;

import java.util.ArrayList;
import java.util.Iterator;

public class Test01_Top10 {
    public static void main(String[] args) throws ClassNotFoundException {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore")
                // 替换默认的序列化机制
                .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
                // 注册需要使用kryo序列化的自定义类
                .registerKryoClasses(new Class[]{Class.forName("com.atguigu.spark.bean.CategoryCountInfo"),Class.forName("com.atguigu.spark.bean.UserVisitAction")});

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 分三次进行wordCount 最后使用cogroup合并三组数据
        JavaRDD<String> lineRDD = sc.textFile("input/user_visit_action.txt");

        // 转换为对象
        JavaRDD<UserVisitAction> actionRDD = lineRDD.map(new Function<String, UserVisitAction>() {
            @Override
            public UserVisitAction call(String v1) throws Exception {
                String[] data = v1.split("_");
                return new UserVisitAction(
                        data[0],
                        data[1],
                        data[2],
                        data[3],
                        data[4],
                        data[5],
                        data[6],
                        data[7],
                        data[8],
                        data[9],
                        data[10],
                        data[11],
                        data[12]
                );
            }
        });

        JavaRDD<CategoryCountInfo> categoryCountInfoJavaRDD = actionRDD.flatMap(new FlatMapFunction<UserVisitAction, CategoryCountInfo>() {
            @Override
            public Iterator<CategoryCountInfo> call(UserVisitAction userVisitAction) throws Exception {
                ArrayList<CategoryCountInfo> countInfos = new ArrayList<>();
                if (!userVisitAction.getClick_category_id().equals("-1")) {
                    // 当前为点击数据
                    countInfos.add(new CategoryCountInfo(userVisitAction.getClick_category_id(), 1L, 0L, 0L));
                } else if (!userVisitAction.getOrder_category_ids().equals("null")) {
                    // 当前为订单数据
                    String[] orders = userVisitAction.getOrder_category_ids().split(",");
                    for (String order : orders) {
                        countInfos.add(new CategoryCountInfo(order, 0L, 1L, 0L));
                    }
                } else if (!userVisitAction.getPay_category_ids().equals("null")) {
                    // 当前为支付数据
                    String[] pays = userVisitAction.getPay_category_ids().split(",");
                    for (String pay : pays) {
                        countInfos.add(new CategoryCountInfo(pay, 0L, 0L, 1L));
                    }
                }
                return countInfos.iterator();
            }
        });

        // 合并相同的id
        JavaPairRDD<String, Iterable<CategoryCountInfo>> groupByRDD = categoryCountInfoJavaRDD.groupBy(new Function<CategoryCountInfo, String>() {
            @Override
            public String call(CategoryCountInfo v1) throws Exception {
                return v1.getCategoryId();
            }
        });

        JavaRDD<CategoryCountInfo> countInfoJavaRDD = groupByRDD.map(new Function<Tuple2<String, Iterable<CategoryCountInfo>>, CategoryCountInfo>() {
            @Override
            public CategoryCountInfo call(Tuple2<String, Iterable<CategoryCountInfo>> v1) throws Exception {
                CategoryCountInfo result = new CategoryCountInfo(v1._1, 0L, 0L, 0L);

                Iterable<CategoryCountInfo> countInfos = v1._2;

                for (CategoryCountInfo countInfo : countInfos) {
                    result.setClickCount(result.getClickCount() + countInfo.getClickCount());
                    result.setOrderCount(result.getOrderCount() + countInfo.getOrderCount());
                    result.setPayCount(result.getPayCount() + countInfo.getPayCount());
                }

                return result;
            }
        });

        // CategoryCountInfo需要能够比较大小
        JavaRDD<CategoryCountInfo> result = countInfoJavaRDD.sortBy(new Function<CategoryCountInfo, CategoryCountInfo>() {
            @Override
            public CategoryCountInfo call(CategoryCountInfo v1) throws Exception {
                return v1;
            }
        }, false, 2);

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

### 7.2.3 需求分析（方案二）算子优化

针对方案二中的 groupBy 算子，没有提前聚合的功能，替换成 reduceByKey。

### 7.2.4 需求实现（方案二）

1）样例类代码和方案一的一样。（详见方案一）
2）核心代码实现

```java
package com.atguigu.spark.demo;

import com.atguigu.spark.bean.CategoryCountInfo;
import com.atguigu.spark.bean.UserVisitAction;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.ArrayList;
import java.util.Iterator;

public class Test06_Top10 {
    public static void main(String[] args) throws ClassNotFoundException {

        // 1.创建配置对象
        SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("sparkCore")
                // 替换默认的序列化机制
                .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
                // 注册需要使用kryo序列化的自定义类
                .registerKryoClasses(new Class[]{Class.forName("com.atguigu.spark.bean.CategoryCountInfo"),Class.forName("com.atguigu.spark.bean.UserVisitAction")});

        // 2. 创建sparkContext
        JavaSparkContext sc = new JavaSparkContext(conf);

        // 3. 编写代码
        // 分三次进行wordCount 最后使用cogroup合并三组数据
        JavaRDD<String> lineRDD = sc.textFile("input/user_visit_action.txt");

        // 转换为对象
        JavaRDD<UserVisitAction> actionRDD = lineRDD.map(new Function<String, UserVisitAction>() {
            @Override
            public UserVisitAction call(String v1) throws Exception {
                String[] data = v1.split("_");
                return new UserVisitAction(
                        data[0],
                        data[1],
                        data[2],
                        data[3],
                        data[4],
                        data[5],
                        data[6],
                        data[7],
                        data[8],
                        data[9],
                        data[10],
                        data[11],
                        data[12]
                );
            }
        });

        JavaRDD<CategoryCountInfo> categoryCountInfoJavaRDD = actionRDD.flatMap(new FlatMapFunction<UserVisitAction, CategoryCountInfo>() {
            @Override
            public Iterator<CategoryCountInfo> call(UserVisitAction userVisitAction) throws Exception {
                ArrayList<CategoryCountInfo> countInfos = new ArrayList<>();
                if (!userVisitAction.getClick_category_id().equals("-1")) {
                    // 当前为点击数据
                    countInfos.add(new CategoryCountInfo(userVisitAction.getClick_category_id(), 1L, 0L, 0L));
                } else if (!userVisitAction.getOrder_category_ids().equals("null")) {
                    // 当前为订单数据
                    String[] orders = userVisitAction.getOrder_category_ids().split(",");
                    for (String order : orders) {
                        countInfos.add(new CategoryCountInfo(order, 0L, 1L, 0L));
                    }
                } else if (!userVisitAction.getPay_category_ids().equals("null")) {
                    // 当前为支付数据
                    String[] pays = userVisitAction.getPay_category_ids().split(",");
                    for (String pay : pays) {
                        countInfos.add(new CategoryCountInfo(pay, 0L, 0L, 1L));
                    }
                }
                return countInfos.iterator();
            }
        });

        // 合并相同的id
        JavaPairRDD<String, CategoryCountInfo> countInfoJavaPairRDD = categoryCountInfoJavaRDD.mapToPair(new PairFunction<CategoryCountInfo, String, CategoryCountInfo>() {
            @Override
            public Tuple2<String, CategoryCountInfo> call(CategoryCountInfo categoryCountInfo) throws Exception {
                return new Tuple2<>(categoryCountInfo.getCategoryId(), categoryCountInfo);
            }
        });

        JavaPairRDD<String, CategoryCountInfo> countInfoPairRDD = countInfoJavaPairRDD.reduceByKey(new Function2<CategoryCountInfo, CategoryCountInfo, CategoryCountInfo>() {
            @Override
            public CategoryCountInfo call(CategoryCountInfo v1, CategoryCountInfo v2) throws Exception {
                v1.setClickCount(v1.getClickCount() + v2.getClickCount());
                v1.setOrderCount(v1.getOrderCount() + v2.getOrderCount());
                v1.setPayCount(v1.getPayCount() + v2.getPayCount());
                return v1;
            }
        });

        JavaRDD<CategoryCountInfo> countInfoJavaRDD = countInfoPairRDD.map(new Function<Tuple2<String, CategoryCountInfo>, CategoryCountInfo>() {
            @Override
            public CategoryCountInfo call(Tuple2<String, CategoryCountInfo> v1) throws Exception {
                return v1._2;
            }
        });

        // CategoryCountInfo需要能够比较大小
        JavaRDD<CategoryCountInfo> result = countInfoJavaRDD.sortBy(new Function<CategoryCountInfo, CategoryCountInfo>() {
            @Override
            public CategoryCountInfo call(CategoryCountInfo v1) throws Exception {
                return v1;
            }
        }, false, 2);

        result. collect().forEach(System.out::println);

        // 4. 关闭sc
        sc.stop();
    }
}
```

