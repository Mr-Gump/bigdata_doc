# 第1章 MapReduce概述

## 1.1 MapReduce定义

MapReduce 是一个{==分布式==}运算程序的编程框架，是用户开发“基于 Hadoop 的数据分析应用”的核心框架。

MapReduce 核心功能是将用户编写的{==业务逻辑代码==}和{==自带默认组件==} 整合成一个完整的分布式运算程序，并发运行在一个 Hadoop 集群上。

## 1.2 MapReduce优缺点

### 1.2.1 优点

- [x] MapReduce 易于编程

它{==简单的实现一些接口==}，就可以完成一个分布式程序，这个分布式程序可以分布到大量廉价的 PC 机器上运行。也就是说你写一个分布式程序，跟写一个简单的串行程序是一模一样的。就是因为这个特点使得 MapReduce 编程变得非常流行。

- [x] 良好的扩展性

当你的计算资源不能得到满足的时候，你可以通过简单的{==增加机器==}来扩展它的计算能力。

- [x] 高容错性

MapReduce 设计的初衷就是使程序能够部署在廉价的 PC 机器上，这就要求它具有很高的容错性。比如{==其中一台机器挂了，它可以把上面的计算任务转移到另外一个节点上运行，不至于这个任务运行失败==}，而且这个过程不需要人工参与，而完全是由 Hadoop 内部完成的。

- [x] 适合 PB 级以上海量数据的离线处理

可以实现上千台服务器集群并发工作，提供数据处理能力。

### 1.2.2 缺点

- [ ] 不擅长实时计算

MapReduce 无法像 MySQL 一样，在毫秒或者秒级内返回结果。

- [ ] 不擅长流式计算

流式计算的输入数据是动态的，而 MapReduce 的{==输入数据集是静态的==}，不能动态变化。这是因为 MapReduce 自身的设计特点决定了数据源必须是静态的。

- [ ] 不擅长 DAG（有向无环图）计算

多个应用程序存在依赖关系，后一个应用程序的输入为前一个的输出。在这种情况下，MapReduce 并不是不能做，而是使用后，{==每个 MapReduce 作业的输出结果都会写入到磁盘，会造成大量的磁盘 IO，导致性能非常的低下==}。

## 1.3 MapReduce核心思想

![image-20230228211226964](https://cos.gump.cloud/uPic/image-20230228211226964.png)

（1）分布式的运算程序往往需要分成至少 2 个阶段。

（2）第一个阶段的 MapTask 并发实例，完全并行运行，互不相干。

（3）第二个阶段的 ReduceTask 并发实例互不相干，但是他们的数据依赖于上一个阶段的所有 MapTask 并发实例的输出。

（4）MapReduce 编程模型只能包含一个 Map 阶段和一个 Reduce 阶段，如果用户的业务逻辑非常复杂，那就只能多个 MapReduce 程序，串行运行。

总结：分析 WordCount 数据流走向深入理解 MapReduce 核心思想。

## 1.4 MapReduce进程

一个完整的 MapReduce 程序在分布式运行时有三类实例进程：

- MrAppMaster：负责整个程序的过程调度及状态协调。
- MapTask：负责 Ma p阶段的整个数据处理流程。
- ReduceTask：负责 Reduce 阶段的整个数据处理流程。

## 1.5 官方WordCount源码

采用反编译工具反编译源码，发现 WordCount 案例有 Map 类、Reduce 类和 Driver 类。且数据的类型是 Hadoop 自身封装的序列化类型。

## 1.6 常用数据序列化类型

| **Java类型** | **Hadoop Writable类型** |
| ------------ | ----------------------- |
| Boolean      | BooleanWritable         |
| Byte         | ByteWritable            |
| Int          | IntWritable             |
| Float        | FloatWritable           |
| Long         | LongWritable            |
| Double       | DoubleWritable          |
| String       | Text                    |
| Map          | MapWritable             |
| Array        | ArrayWritable           |
| Null         | NullWritable            |

## 1.7 MapReduce编程规范

用户编写的程序分成三个部分：Mapper、Reducer 和 Driver。

![image-20230228211527473](https://cos.gump.cloud/uPic/image-20230228211527473.png)

![image-20230228211544261](https://cos.gump.cloud/uPic/image-20230228211544261.png)

## 1.8 WordCount案例实操

### 1.8.1 本地测试

**1）需求**

在给定的文本文件中统计输出每一个单词出现的总次数。

（1）输入数据

``` title="hello.txt" 
--8<-- "hello.txt"
```

（2）期望输出数据

```text
atguigu	2
banzhang	1
cls	2
hadoop	1
jiao	1
ss	2
xue	1
```

**2）需求分析**

按照 MapReduce 编程规范，分别编写 Mapper，Reducer，Driver。

![image-20230228211931786](https://cos.gump.cloud/uPic/image-20230228211931786.png)

**3）环境准备**

（1）创建 maven 工程，MapReduceDemo

（2）在 pom.xml 文件中添加如下依赖

```xml title="pom.xml"
<dependencies>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>3.1.3</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
        <version>1.7.30</version>
    </dependency>
</dependencies>
```

（2）在项目的 src/main/resources 目录下，新建一个文件，命名为 `log4j.properties`，在文件中填入。

```properties title="log4j.properties"
log4j.rootLogger=INFO, stdout  
log4j.appender.stdout=org.apache.log4j.ConsoleAppender  
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout  
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n  
log4j.appender.logfile=org.apache.log4j.FileAppender  
log4j.appender.logfile.File=target/spring.log  
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout  
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```

（3）创建包名：com.atguigu.mapreduce.wordcount

**4）编写程序**

（1）编写 Mapper 类

```java
package com.atguigu.mapreduce.wordcount;
import java.io.IOException;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
	
	Text k = new Text();
	IntWritable v = new IntWritable(1);
	
	@Override
	protected void map(LongWritable key, Text value, Context context)	throws IOException, InterruptedException {
		
		// 1 获取一行
		String line = value.toString();
		
		// 2 切割
		String[] words = line.split(" ");
		
		// 3 输出
		for (String word : words) {
			
			k.set(word);
			context.write(k, v);
		}
	}
}
```

（2）编写 Reducer 类

```java
package com.atguigu.mapreduce.wordcount;
import java.io.IOException;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable>{

int sum;
IntWritable v = new IntWritable();

	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException {
		
		// 1 累加求和
		sum = 0;
		for (IntWritable count : values) {
			sum += count.get();
		}
		
		// 2 输出
         v.set(sum);
		context.write(key,v);
	}
}
```

（3）编写 Driver 驱动类

```shell
package com.atguigu.mapreduce.wordcount;
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCountDriver {

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

		// 1 获取配置信息以及获取job对象
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

		// 2 关联本Driver程序的jar
		job.setJarByClass(WordCountDriver.class);

		// 3 关联Mapper和Reducer的jar
		job.setMapperClass(WordCountMapper.class);
		job.setReducerClass(WordCountReducer.class);

		// 4 设置Mapper输出的kv类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);

		// 5 设置最终输出kv类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		
		// 6 设置输入和输出路径
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		// 7 提交job
		boolean result = job.waitForCompletion(true);
		System.exit(result ? 0 : 1);
	}
}
```

**5）本地测试**

（1）需要首先配置好 HADOOP_HOME 变量以及 Windows 运行依赖

（2）在 IDEA / Eclipse 上运行程序

### 1.8.2 提交到集群测试

**1）集群上测试**

（1）用 maven 打 jar 包，需要添加的打包插件依赖

```xml title="pom.xml"
<build>
    <plugins>
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.6.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

!!! info "注意"

    如果工程上显示红叉。在项目上右键 -> maven -> Reimport 刷新即可。

**（2）将程序打成 jar 包**

![image-20230228212258216](https://cos.gump.cloud/uPic/image-20230228212258216.png)

（3）修改不带依赖的 jar 包名称为 wc.jar，并拷贝该 jar 包到 Hadoop 集群的 /opt/module/hadoop-3.1.3 路径。

（4）启动 Hadoop 集群

```shell
sbin/start-dfs.sh
sbin/start-yarn.sh
```

（5）执行 WordCount 程序

```shell
hadoop jar  wc.jar com.atguigu.mapreduce.wordcount.WordCountDriver /user/atguigu/input /user/atguigu/output
```

