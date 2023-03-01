# 第3章 MapReduce框架原理

![image-20230301081012107](https://cos.gump.cloud/uPic/image-20230301081012107.png)

## 3.1 InputFormat数据输入

### 3.1.1 切片与MapTask并行度决定机制

**1）问题引出**

MapTask 的并行度决定 Map 阶段的任务处理并发度，进而影响到整个 Job 的处理速度。

!!! question "思考"

    1G 的数据，启动 8 个 MapTask，可以提高集群的并发处理能力。那么 1K 的数据，也启动 8 个 MapTask，会提高集群性能吗？MapTask 并行任务是否越多越好呢？哪些因素影响了 MapTask 并行度？

**2）MapTask 并行度决定机制**

- 数据块：Block 是 HDFS 物理上把数据分成一块一块。{==数据块是 HDFS 存储数据单位==}。
- 数据切片：数据切片只是在逻辑上对输入进行分片，并不会在磁盘上将其切分成片进行存储。{==数据切片是 MapReduce 程序计算输入数据的单位==}，一个切片会对应启动一个 MapTask。

![image-20230301081235849](https://cos.gump.cloud/uPic/image-20230301081235849.png)

### 3.1.2 Job提交流程源码和切片源码详解

**1）Job 提交流程源码详解**

```java
waitForCompletion()

submit();

// 1建立连接
	connect();	
		// 1）创建提交Job的代理
		new Cluster(getConfiguration());
			// （1）判断是本地运行环境还是yarn集群运行环境
			initialize(jobTrackAddr, conf); 

// 2 提交job
submitter.submitJobInternal(Job.this, cluster)

	// 1）创建给集群提交数据的Stag路径
	Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);

	// 2）获取jobid ，并创建Job路径
	JobID jobId = submitClient.getNewJobID();

	// 3）拷贝jar包到集群
copyAndConfigureFiles(job, submitJobDir);	
	rUploader.uploadFiles(job, jobSubmitDir);

	// 4）计算切片，生成切片规划文件
writeSplits(job, submitJobDir);
		maps = writeNewSplits(job, jobSubmitDir);
		input.getSplits(job);

	// 5）向Stag路径写XML配置文件
writeConf(conf, submitJobFile);
	conf.writeXml(out);

	// 6）提交Job,返回提交状态
status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());
```

<figure markdown>
  ![image-20230301081339230](https://cos.gump.cloud/uPic/image-20230301081339230.png)
  <figcaption>Job提交流程源码分析</figcaption>
</figure>

**2）FileInputFormat 切片源码解析（`input.getSplits(job)`）**

![image-20230301081703398](https://cos.gump.cloud/uPic/image-20230301081703398.png)

### 3.1.3 FileInputFormat切片机制

<figure markdown>
  ![image-20230301081808794](https://cos.gump.cloud/uPic/image-20230301081808794.png)
  <figcaption>FileInputFormat切片机制</figcaption>
</figure>
<figure markdown>
  ![image-20230301081902494](https://cos.gump.cloud/uPic/image-20230301081902494.png)
  <figcaption>FileInputFormat切片大小的参数配置</figcaption>
</figure>

### 3.1.4 TextInputFormat

**1）FileInputFormat 实现类**

!!! question "思考"

    在运行 MapReduce 程序时，输入的文件格式包括：{++基于行的日志文件++}、{++二进制格式文件++}、{++数据库表++}等。那么，针对不同的数据类型，MapReduce 是如何读取这些数据的呢？

FileInputFormat 常见的接口实现类包括：{++TextInputFormat++}、{++KeyValueTextInputFormat++}、{++NLineInputFormat++}、{++CombineTextInputFormat++} 和{++自定义 InputFormat++} 等。

**2）TextInputFormat**

TextInputFormat 是默认的 FileInputFormat 实现类。按行读取每条记录。{==键是存储该行在整个文件中的起始字节偏移量， LongWritable 类型。值是这行的内容，不包括任何行终止符（换行符和回车符）==}，Text 类型。

以下是一个示例，比如，一个分片包含了如下 4 条文本记录。

```shell
Rich learning form
Intelligent learning engine
Learning more convenient
From the real demand for more close to the enterprise
```

每条记录表示为以下键/值对：

```shell
(0,Rich learning form)
(20,Intelligent learning engine)
(49,Learning more convenient)
(74,From the real demand for more close to the enterprise)
```

### 3.1.5 CombineTextInputFormat切片机制

框架默认的 TextInputFormat 切片机制是对任务按文件规划切片，{==不管文件多小，都会是一个单独的切片==}，都会交给一个 MapTask，这样如果有{==大量小文件==}，就会产生大量的 MapTask，处理效率极其低下。

**1）应用场景**

CombineTextInputFormat 用于小文件过多的场景，它可以将多个小文件从逻辑上规划到一个切片中，这样，多个小文件就可以交给一个 MapTask 处理。

**2）虚拟存储切片最大值设置**

`CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);// 4m`

!!! info "注意"

    虚拟存储切片最大值设置最好根据实际的小文件大小情况来设置具体的值。

**3）切片机制**

生成切片过程包括：{++虚拟存储++}过程和{++切片++}过程二部分。

![image-20230301181121488](https://cos.gump.cloud/uPic/image-20230301181121488.png)

（1）虚拟存储过程：

将输入目录下所有文件大小，依次和设置的 setMaxInputSplitSize 值比较，如果不大于设置的最大值，逻辑上划分一个块。如果输入文件大于设置的最大值且大于两倍，那么以最大值切割一块；{==当剩余数据大小超过设置的最大值且不大于最大值 2 倍，此时将文件均分成 2 个虚拟存储块（防止出现太小切片）==}。

例如 setMaxInputSplitSize 值为 4M，输入文件大小为 8.02 M，则先逻辑上 分成一个 4M。剩余的大小为 4.02M，如果按照 4M 逻辑划分，就会出现 0.02M 的小的虚拟存储文件，所以将剩余的 4.02M 文件切分成（2.01M 和 2.01M）两个文件。

（2）切片过程：

①判断虚拟存储的文件大小是否大于 setMaxInputSplitSize 值，大于等于则单独形成一个切片。

②如果不大于则跟下一个虚拟存储文件进行合并，共同形成一个切片。

③测试举例：有 4 个小文件大小分别为 1.7M、5.1M、3.4M 以及 6.8M 这四个小文件，则虚拟存储之后形成 6 个文件块，大小分别为：
1.7M，（2.55M、2.55M），3.4M以及（3.4M、3.4M）
最终会形成 3 个切片，大小分别为：
（1.7 + 2.55）M，（2.55 + 3.4M ，（3.4 + 3.4）M

### 3.1.6 CombineTextInputFormat案例实操

**1）需求**

将输入的大量小文件合并成一个切片统一处理。

（1）输入数据

准备 4 个小文件

（2）期望

期望一个切片处理 4 个文件

**2）实现过程**

（1）不做任何处理，运行 1.8 节的 WordCount 案例程序，观察切片个数为 4。

```shell
number of splits:4
```

（2）在 WordcountDriver 中增加如下代码，运行程序，并观察运行的切片个数为 3。

①驱动类中添加代码如下：

```java
// 如果不设置InputFormat，它默认用的是TextInputFormat.class
job.setInputFormatClass(CombineTextInputFormat.class);

//虚拟存储切片最大值设置4m
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);
```

​	②运行如果为 3 个切片。

```java
number of splits:3
```

（3）在 WordcountDriver 中增加如下代码，运行程序，并观察运行的切片个数为 1。

①驱动中添加代码如下：

```java
// 如果不设置InputFormat，它默认用的是TextInputFormat.class
job.setInputFormatClass(CombineTextInputFormat.class);

//虚拟存储切片最大值设置20m
CombineTextInputFormat.setMaxInputSplitSize(job, 20971520);
```

②运行如果为 1 个切片。

```shell
number of splits:1
```

## 3.2 MapReduce工作流程

![image-20230301181948641](https://cos.gump.cloud/uPic/image-20230301181948641.png)

![image-20230301182159661](https://cos.gump.cloud/uPic/image-20230301182159661.png)

上面的流程是整个 MapReduce 最全工作流程，但是 Shuffle 过程只是从第 7 步开始到第 16 步结束，具体 Shuffle 过程详解，如下：

（1）MapTask 收集我们的 map() 方法输出的 kv 对，放到内存缓冲区中

（2）从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件

（3）多个溢出文件会被合并成大的溢出文件

（4）在溢出过程及合并的过程中，都要调用 Partitioner 进行分区和针对 key 进行排序

（5）ReduceTask 根据自己的分区号，去各个 MapTask 机器上取相应的结果分区数据

（6）ReduceTask 会抓取到同一个分区的来自不同 MapTask 的结果文件，ReduceTask 会将这些文件再进行合并（归并排序）

（7）合并成大文件后，Shuffle 的过程也就结束了，后面进入 ReduceTask 的逻辑运算过程（从文件中取出一个一个的键值对 Group，调用用户自定义的 reduce() 方法）

!!! info "注意"

    （1）Shuffle 中的缓冲区大小会影响到 MapReduce 程序的执行效率，原则上说，缓冲区越大，磁盘 io 的次数越少，执行速度就越快。
    
    （2）缓冲区的大小可以通过参数调整，参数：`mapreduce.task.io.sort.mb` 默认 100M。

## 3.3 Shuffle机制

### 3.3.1 Shuffle机制

Map 方法之后，Reduce 方法之前的数据处理过程称之为 Shuffle。

![image-20230301182557524](https://cos.gump.cloud/uPic/image-20230301182557524.png)

### 3.3.2 Partition分区

![image-20230301182808443](https://cos.gump.cloud/uPic/image-20230301182808443.png)

![image-20230301182817828](https://cos.gump.cloud/uPic/image-20230301182817828.png)

![image-20230301182828766](https://cos.gump.cloud/uPic/image-20230301182828766.png)

### 3.3.3 Partition分区案例实操

**1）需求**

将统计结果按照手机归属地不同省份输出到不同文件中（分区）

（1）输入数据

``` title="phone_data.txt" 
--8<-- "phone_data.txt"
```

（2）期望输出数据

​	手机号 136、137、138、139 开头都分别放到一个独立的 4 个文件中，其他开头的放到一个文件中。

**2）需求分析**

![image-20230301183018063](https://cos.gump.cloud/uPic/image-20230301183018063.png)

**3）在案例 2.3 的基础上，增加一个分区类**

```java
package com.atguigu.mapreduce.partitioner;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

public class ProvincePartitioner extends Partitioner<Text, FlowBean> {

    @Override
    public int getPartition(Text text, FlowBean flowBean, int numPartitions) {
        //获取手机号前三位prePhone
        String phone = text.toString();
        String prePhone = phone.substring(0, 3);

        //定义一个分区号变量partition,根据prePhone设置分区号
        int partition;

        if("136".equals(prePhone)){
            partition = 0;
        }else if("137".equals(prePhone)){
            partition = 1;
        }else if("138".equals(prePhone)){
            partition = 2;
        }else if("139".equals(prePhone)){
            partition = 3;
        }else {
            partition = 4;
        }

        //最后返回分区号partition
        return partition;
    }
}
```

**4）在驱动函数中增加自定义数据分区设置和 ReduceTask 设置**

```java
package com.atguigu.mapreduce.partitioner;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class FlowDriver {

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

        //1 获取job对象
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        //2 关联本Driver类
        job.setJarByClass(FlowDriver.class);

        //3 关联Mapper和Reducer
        job.setMapperClass(FlowMapper.class);
        job.setReducerClass(FlowReducer.class);

        //4 设置Map端输出数据的KV类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(FlowBean.class);

        //5 设置程序最终输出的KV类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(FlowBean.class);

        //8 指定自定义分区器
        job.setPartitionerClass(ProvincePartitioner.class);

        //9 同时指定相应数量的ReduceTask
        job.setNumReduceTasks(5);

        //6 设置输入输出路径
        FileInputFormat.setInputPaths(job, new Path("D:\\inputflow"));
        FileOutputFormat.setOutputPath(job, new Path("D\\partitionout"));

        //7 提交Job
        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

### 3.3.4 WritableComparable排序

![image-20230301183120645](https://cos.gump.cloud/uPic/image-20230301183120645.png)

![image-20230301183135843](https://cos.gump.cloud/uPic/image-20230301183135843.png)

<figure markdown>
  ![image-20230301183211731](https://cos.gump.cloud/uPic/image-20230301183211731.png)
  <figcaption>排序分类</figcaption>
</figure>

**自定义排序 WritableComparable 原理分析**

bean 对象做为 key 传输，需要实现 `WritableComparable` 接口重写 compareTo 方法，就可以实现排序。

```java
@Override
public int compareTo(FlowBean bean) {

	int result;
		
	// 按照总流量大小，倒序排列
	if (this.sumFlow > bean.getSumFlow()) {
		result = -1;
	}else if (this.sumFlow < bean.getSumFlow()) {
		result = 1;
	}else {
		result = 0;
	}

	return result;
}
```

### 3.3.5 WritableComparable排序案例实操（全排序）

**1）需求**

根据案例 2.3 序列化案例产生的结果再次对总流量进行倒序排序。

（1）输入数据

（2）期望输出数据

```shell
13509468723	7335	110349	117684
13736230513	2481	24681	27162
13956435636	132		1512	1644
13846544121	264		0		264
。。。 。。。
```

**2）需求分析**

![image-20230301183416337](https://cos.gump.cloud/uPic/image-20230301183416337.png)

**3）代码实现**

（1）FlowBean 对象在在需求 1 基础上增加了比较功能

```java
package com.atguigu.mapreduce.writablecompable;

import org.apache.hadoop.io.WritableComparable;
import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class FlowBean implements WritableComparable<FlowBean> {

    private long upFlow; //上行流量
    private long downFlow; //下行流量
    private long sumFlow; //总流量

    //提供无参构造
    public FlowBean() {
    }

    //生成三个属性的getter和setter方法
    public long getUpFlow() {
        return upFlow;
    }

    public void setUpFlow(long upFlow) {
        this.upFlow = upFlow;
    }

    public long getDownFlow() {
        return downFlow;
    }

    public void setDownFlow(long downFlow) {
        this.downFlow = downFlow;
    }

    public long getSumFlow() {
        return sumFlow;
    }

    public void setSumFlow(long sumFlow) {
        this.sumFlow = sumFlow;
    }

    public void setSumFlow() {
        this.sumFlow = this.upFlow + this.downFlow;
    }

    //实现序列化和反序列化方法,注意顺序一定要一致
    @Override
    public void write(DataOutput out) throws IOException {
        out.writeLong(this.upFlow);
        out.writeLong(this.downFlow);
        out.writeLong(this.sumFlow);

    }

    @Override
    public void readFields(DataInput in) throws IOException {
        this.upFlow = in.readLong();
        this.downFlow = in.readLong();
        this.sumFlow = in.readLong();
    }

    //重写ToString,最后要输出FlowBean
    @Override
    public String toString() {
        return upFlow + "\t" + downFlow + "\t" + sumFlow;
    }

    @Override
    public int compareTo(FlowBean o) {

        //按照总流量比较,倒序排列
        if(this.sumFlow > o.sumFlow){
            return -1;
        }else if(this.sumFlow < o.sumFlow){
            return 1;
        }else {
            return 0;
        }
    }
}
```

（2）编写 Mapper 类

```java
package com.atguigu.mapreduce.writablecompable;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;

public class FlowMapper extends Mapper<LongWritable, Text, FlowBean, Text> {
    private FlowBean outK = new FlowBean();
    private Text outV = new Text();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        //1 获取一行数据
        String line = value.toString();

        //2 按照"\t",切割数据
        String[] split = line.split("\t");

        //3 封装outK outV
        outK.setUpFlow(Long.parseLong(split[1]));
        outK.setDownFlow(Long.parseLong(split[2]));
        outK.setSumFlow();
        outV.set(split[0]);

        //4 写出outK outV
        context.write(outK,outV);
    }
}
```

（3）编写 Reducer 类

```java
package com.atguigu.mapreduce.writablecompable;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class FlowReducer extends Reducer<FlowBean, Text, Text, FlowBean> {
    @Override
    protected void reduce(FlowBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {

        //遍历values集合,循环写出,避免总流量相同的情况
        for (Text value : values) {
            //调换KV位置,反向写出
            context.write(value,key);
        }
    }
}
```

（4）编写 Driver 类

```java
package com.atguigu.mapreduce.writablecompable;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class FlowDriver {

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

        //1 获取job对象
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        //2 关联本Driver类
        job.setJarByClass(FlowDriver.class);

        //3 关联Mapper和Reducer
        job.setMapperClass(FlowMapper.class);
        job.setReducerClass(FlowReducer.class);

        //4 设置Map端输出数据的KV类型
        job.setMapOutputKeyClass(FlowBean.class);
        job.setMapOutputValueClass(Text.class);

        //5 设置程序最终输出的KV类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(FlowBean.class);

        //6 设置输入输出路径
        FileInputFormat.setInputPaths(job, new Path("D:\\inputflow2"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\comparout"));

        //7 提交Job
        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

### 3.3.6 WritableComparable排序案例实操（区内排序）

**1）需求**

要求每个省份手机号输出的文件中按照总流量内部排序。

**2）需求分析**

基于前一个需求，增加自定义分区类，分区按照省份手机号设置。

![image-20230301183617163](https://cos.gump.cloud/uPic/image-20230301183617163.png)

3）案例实操

（1）增加自定义分区类

```java
package com.atguigu.mapreduce.partitionercompable;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

public class ProvincePartitioner2 extends Partitioner<FlowBean, Text> {

    @Override
    public int getPartition(FlowBean flowBean, Text text, int numPartitions) {
        //获取手机号前三位
        String phone = text.toString();
        String prePhone = phone.substring(0, 3);

        //定义一个分区号变量partition,根据prePhone设置分区号
        int partition;
        if("136".equals(prePhone)){
            partition = 0;
        }else if("137".equals(prePhone)){
            partition = 1;
        }else if("138".equals(prePhone)){
            partition = 2;
        }else if("139".equals(prePhone)){
            partition = 3;
        }else {
            partition = 4;
        }

        //最后返回分区号partition
        return partition;
    }
}
```

（2）在驱动类中添加分区类

```java
// 设置自定义分区器
job.setPartitionerClass(ProvincePartitioner2.class);

// 设置对应的ReduceTask的个数
job.setNumReduceTasks(5);
```

### 3.3.7 Combiner合并

![image-20230301183706911](https://cos.gump.cloud/uPic/image-20230301183706911.png)

（6）自定义 Combiner 实现步骤

①自定义一个 Combiner 继承 Reducer，重写 Reduce 方法

```java
public class WordCountCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {

    private IntWritable outV = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {

        int sum = 0;
        for (IntWritable value : values) {
            sum += value.get();
        }
     
        outV.set(sum);
     
        context.write(key,outV);
    }
}
```

②在 Job 驱动类中设置

```java
job.setCombinerClass(WordCountCombiner.class);
```

### 3.3.8 Combiner合并案例实操

**1）需求**

统计过程中对每一个 MapTask 的输出进行局部汇总，以减小网络传输量即采用 Combiner 功能。

（1）数据输入

``` title="phone_data.txt" 
--8<-- "hello.txt"
```

（2）期望输出数据

期望：Combine 输入数据多，输出时经过合并，输出数据降低。

**2）需求分析**

![image-20230301183914753](https://cos.gump.cloud/uPic/image-20230301183914753.png)

**3）案例实操-方案一**

（1）增加一个 WordCountCombiner 类继承 Reducer

```java
ackage com.atguigu.mapreduce.combiner;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class WordCountCombiner extends Reducer<Text, IntWritable, Text, IntWritable> {

private IntWritable outV = new IntWritable();

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {

        int sum = 0;
        for (IntWritable value : values) {
            sum += value.get();
        }

        //封装outKV
        outV.set(sum);

        //写出outKV
        context.write(key,outV);
    }
}
```

（2）在 WordcountDriver 驱动类中指定 Combiner

```java
// 指定需要使用combiner，以及用哪个类作为combiner的逻辑
job.setCombinerClass(WordCountCombiner.class);
```

**4）案例实操-方案二**

（1）将 WordcountReducer 作为 Combiner 在 WordcountDriver 驱动类中指定

```java
// 指定需要使用Combiner，以及用哪个类作为Combiner的逻辑
job.setCombinerClass(WordCountReducer.class);
```

运行程序，如下图所示

=== "使用前"

    ![image-20230301184039607](https://cos.gump.cloud/uPic/image-20230301184039607.png)

=== "使用后"

    ![image-20230301184048314](https://cos.gump.cloud/uPic/image-20230301184048314.png)

## 3.4 OutputFormat数据输出

### 3.4.1 OutputFormat接口实现类

![image-20230301184142854](https://cos.gump.cloud/uPic/image-20230301184142854.png)

### 3.4.2 自定义OutputFormat案例实操

**1）需求**

​	过滤输入的 log 日志，包含 atguigu 的网站输出到 e:/atguigu.log，不包含 atguigu 的网站输出到 e:/other.log。

（1）输入数据

``` title="hello.txt" 
--8<-- "log.txt"
```

（2）期望输出数据

包含 atguigu ===> atguigu.txt 其他 ==> other.txt

**2）需求分析**

![image-20230301184448538](https://cos.gump.cloud/uPic/image-20230301184448538.png)

**3）案例实操**

（1）编写 LogMapper 类

```java
package com.atguigu.mapreduce.outputformat;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

public class LogMapper extends Mapper<LongWritable, Text,Text, NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //不做任何处理,直接写出一行log数据
        context.write(value,NullWritable.get());
    }
}
```

（2）编写 LogReducer 类

```java
package com.atguigu.mapreduce.outputformat;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class LogReducer extends Reducer<Text, NullWritable,Text, NullWritable> {
    @Override
    protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
        // 防止有相同的数据,迭代写出
        for (NullWritable value : values) {
            context.write(key,NullWritable.get());
        }
    }
}
```

（3）自定义一个 LogOutputFormat 类

```java
package com.atguigu.mapreduce.outputformat;

import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class LogOutputFormat extends FileOutputFormat<Text, NullWritable> {
    @Override
    public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext job) throws IOException, InterruptedException {
        //创建一个自定义的RecordWriter返回
        LogRecordWriter logRecordWriter = new LogRecordWriter(job);
        return logRecordWriter;
    }
}
```

（4）编写 LogRecordWriter 类

```java
package com.atguigu.mapreduce.outputformat;

import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;

import java.io.IOException;

public class LogRecordWriter extends RecordWriter<Text, NullWritable> {

    private FSDataOutputStream atguiguOut;
    private FSDataOutputStream otherOut;

    public LogRecordWriter(TaskAttemptContext job) {
        try {
            //获取文件系统对象
            FileSystem fs = FileSystem.get(job.getConfiguration());
            //用文件系统对象创建两个输出流对应不同的目录
            atguiguOut = fs.create(new Path("d:/hadoop/atguigu.log"));
            otherOut = fs.create(new Path("d:/hadoop/other.log"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void write(Text key, NullWritable value) throws IOException, InterruptedException {
        String log = key.toString();
        //根据一行的log数据是否包含atguigu,判断两条输出流输出的内容
        if (log.contains("atguigu")) {
            atguiguOut.writeBytes(log + "\n");
        } else {
            otherOut.writeBytes(log + "\n");
        }
    }

    @Override
    public void close(TaskAttemptContext context) throws IOException, InterruptedException {
        //关流
        IOUtils.closeStream(atguiguOut);
        IOUtils.closeStream(otherOut);
    }
}
```

（5）编写 LogDriver 类

```java
package com.atguigu.mapreduce.outputformat;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class LogDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        job.setJarByClass(LogDriver.class);
        job.setMapperClass(LogMapper.class);
        job.setReducerClass(LogReducer.class);

        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(NullWritable.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        //设置自定义的outputformat
        job.setOutputFormatClass(LogOutputFormat.class);

        FileInputFormat.setInputPaths(job, new Path("D:\\input"));
        //虽然我们自定义了outputformat，但是因为我们的outputformat继承自fileoutputformat
        //而fileoutputformat要输出一个_SUCCESS文件，所以在这还得指定一个输出目录
        FileOutputFormat.setOutputPath(job, new Path("D:\\logoutput"));

        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

## 3.5 MapReduce内核源码解析

### 3.5.1 MapTask工作机制

![image-20230301184628079](https://cos.gump.cloud/uPic/image-20230301184628079.png)

（1）Read 阶段：MapTask 通过 InputFormat 获得的 RecordReader，从输入 InputSplit 中解析出一个个 key/value。

（2）Map 阶段：该节点主要是将解析出的 key/value 交给用户编写 map() 函数处理，并产生一系列新的 key/value。

（3）Collect 收集阶段：在用户编写 map() 函数中，当数据处理完成后，一般会调用 OutputCollector.collect() 输出结果。在该函数内部，它会将生成的 key/value 分区（调用 Partitioner），并写入一个环形内存缓冲区中。

（4）Spill 阶段：即“溢写”，当环形缓冲区满后，MapReduce 会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。

​	溢写阶段详情：

步骤 1：利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号 Partition 进行排序，然后按照 key 进行排序。这样，经过排序后，数据以分区为单位聚集在一起，且同一分区内所有数据按照 key 有序。

步骤 2：按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件 output / spillN.out（N 表示当前溢写次数）中。如果用户设置了 Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。

步骤 3：将分区数据的元信息写到内存索引数据结构 SpillRecord 中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过 1MB，则将内存索引写到文件 output / spillN.out.index 中。

（5）Merge 阶段：当所有数据处理完成后，MapTask 对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。

当所有数据处理完后，MapTask 会将所有临时文件合并成一个大文件，并保存到文件 output / file.out 中，同时生成相应的索引文件 output / file.out.index。

在进行文件合并过程中，MapTask 以分区为单位进行合并。对于某个分区，它将采用多轮递归合并的方式。每轮合并`mapreduce.task.io.sort.factor`（默认 10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

让每个 MapTask 最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。

### 3.5.2 ReduceTask工作机制

![image-20230301184948153](https://cos.gump.cloud/uPic/image-20230301184948153.png)

（1）Copy 阶段：ReduceTask 从各个 MapTask 上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，否则直接放到内存中。

（2）Sort 阶段：在远程拷贝数据的同时，ReduceTask 启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多。按照 MapReduce 语义，用户编写 reduce() 函数输入数据是按 key 进行聚集的一组数据。为了将 key 相同的数据聚在一起，Hadoop 采用了基于排序的策略。由于各个 MapTask 已经实现对自己的处理结果进行了局部排序，因此，ReduceTask 只需对所有数据进行一次归并排序即可。
（3）Reduce 阶段：reduce() 函数将计算结果写到 HDFS 上。

### 3.5.3 ReduceTask并行度决定机制

!!! qeustion "思考"

    MapTask 并行度由切片个数决定，切片个数由输入文件和切片规则决定。
    
    ReduceTask 并行度由谁决定？

**1）设置 ReduceTask 并行度（个数）**

ReduceTask 的并行度同样影响整个 Job 的执行并发度和执行效率，但与 MapTask 的并发数由切片数决定不同，ReduceTask 数量的决定是可以直接手动设置：

```java
// 默认值是1，手动设置为4
job.setNumReduceTasks(4);
```

**2）实验：测试 ReduceTask 多少合适**

（1）实验环境：1 个 Master 节点，16 个 Worker 节点：CPU:8GHZ，内存: 2G

（2）实验结论：

| MapTask =16 |      |      |      |      |      |      |      |      |      |      |
| ----------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| ReduceTask  | 1    | 5    | 10   | 15   | 16   | 20   | 25   | 30   | 45   | 60   |
| 总时间      | 892  | 146  | 110  | 92   | 88   | 100  | 128  | 101  | 145  | 104  |

**3）注意事项**

![image-20230301185308958](https://cos.gump.cloud/uPic/image-20230301185308958.png)

### 3.5.4 MapTask & ReduceTask源码解析

**1）MapTask 源码解析流程**

```java
=================== MapTask ===================
context.write(k, NullWritable.get());   //自定义的map方法的写出，进入
output.write(key, value);  
//MapTask727行，收集方法，进入两次 
collector.collect(key, value,partitioner.getPartition(key, value, partitions));
HashPartitioner(); //默认分区器
collect()  //MapTask1082行 map端所有的kv全部写出后会走下面的close方法
close() //MapTask732行
collector.flush() // 溢出刷写方法，MapTask735行，提前打个断点，进入
sortAndSpill() //溢写排序，MapTask1505行，进入
sorter.sort()   QuickSort //溢写排序方法，MapTask1625行，进入
mergeParts(); //合并文件，MapTask1527行，进入	
collector.close(); //MapTask739行,收集器关闭,即将进入ReduceTask
```

**2）ReduceTask 源码解析流程**

```java
=================== ReduceTask ===================
if (isMapOrReduce())  //reduceTask324行，提前打断点
initialize()   // reduceTask333行,进入
init(shuffleContext);  // reduceTask375行,走到这需要先给下面的打断点
totalMaps = job.getNumMapTasks(); // ShuffleSchedulerImpl第120行，提前打断点
merger = createMergeManager(context); //合并方法，Shuffle第80行
// MergeManagerImpl第232 235行，提前打断点
this.inMemoryMerger = createInMemoryMerger(); //内存合并
this.onDiskMerger = new OnDiskMerger(this); //磁盘合并
rIter = shuffleConsumerPlugin.run();
eventFetcher.start();  //开始抓取数据，Shuffle第107行，提前打断点
eventFetcher.shutDown();  //抓取结束，Shuffle第141行，提前打断点
copyPhase.complete();   //copy阶段完成，Shuffle第151行
taskStatus.setPhase(TaskStatus.Phase.SORT);  //开始排序阶段，Shuffle第152行
sortPhase.complete();   //排序阶段完成，即将进入reduce阶段 reduceTask382行
reduce();  //reduce阶段调用的就是我们自定义的reduce方法，会被调用多次
cleanup(context); //reduce完成之前，会最后调用一次Reducer里面的cleanup方法
```

## 3.6 Join应用

### 3.6.1 Reduce Join

Map 端的主要工作：为来自不同表或文件的 key / value 对，{==打标签以区别不同来源的记录==}。然后{==用连接字段作为 key==}，其余部分和新加的标志作为 value，最后进行输出。

 Reduce 端的主要工作：在 Reduce 端{==以连接字段作为 key 的分组已经完成==}，我们只需要在每一个分组当中将那些来源于不同文件的记录（在 Map 阶段已经打标志）分开，最后进行合并就 ok 了。

### 3.6.2 Reduce Join案例实操

**1）需求**

| id   | pid  | amount |
| ---- | ---- | ------ |
| 1001 | 01   | 1      |
| 1002 | 02   | 2      |
| 1003 | 03   | 3      |
| 1004 | 01   | 4      |
| 1005 | 02   | 5      |
| 1006 | 03   | 6      |

| pid  | pname |
| ---- | ----- |
| 01   | 小米  |
| 02   | 华为  |
| 03   | 格力  |

​	将商品信息表中数据根据商品 pid 合并到订单数据表中。

| id   | pname | amount |
| ---- | ----- | ------ |
| 1001 | 小米  | 1      |
| 1004 | 小米  | 4      |
| 1002 | 华为  | 2      |
| 1005 | 华为  | 5      |
| 1003 | 格力  | 3      |
| 1006 | 格力  | 6      |

**2）需求分析**

通过将关联条件作为 Map 输出的 key，将两表满足 Join 条件的数据并携带数据所来源的文件信息，发往同一个 ReduceTask，在 Reduce 中进行数据的串联。

![image-20230301190111197](https://cos.gump.cloud/uPic/image-20230301190111197.png)

**3）代码实现**

（1）创建商品和订单合并后的 TableBean 类

```java
package com.atguigu.mapreduce.reducejoin;

import org.apache.hadoop.io.Writable;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class TableBean implements Writable {

    private String id; //订单id
    private String pid; //产品id
    private int amount; //产品数量
    private String pname; //产品名称
    private String flag; //判断是order表还是pd表的标志字段

    public TableBean() {
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getPid() {
        return pid;
    }

    public void setPid(String pid) {
        this.pid = pid;
    }

    public int getAmount() {
        return amount;
    }

    public void setAmount(int amount) {
        this.amount = amount;
    }

    public String getPname() {
        return pname;
    }

    public void setPname(String pname) {
        this.pname = pname;
    }

    public String getFlag() {
        return flag;
    }

    public void setFlag(String flag) {
        this.flag = flag;
    }

    @Override
    public String toString() {
        return id + "\t" + pname + "\t" + amount;
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(id);
        out.writeUTF(pid);
        out.writeInt(amount);
        out.writeUTF(pname);
        out.writeUTF(flag);
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        this.id = in.readUTF();
        this.pid = in.readUTF();
        this.amount = in.readInt();
        this.pname = in.readUTF();
        this.flag = in.readUTF();
    }
}
```

（2）编写 TableMapper 类

```java
package com.atguigu.mapreduce.reducejoin;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;

import java.io.IOException;

public class TableMapper extends Mapper<LongWritable,Text,Text,TableBean> {

    private String filename;
    private Text outK = new Text();
    private TableBean outV = new TableBean();

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        //获取对应文件名称
        InputSplit split = context.getInputSplit();
        FileSplit fileSplit = (FileSplit) split;
        filename = fileSplit.getPath().getName();
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        //获取一行
        String line = value.toString();

        //判断是哪个文件,然后针对文件进行不同的操作
        if(filename.contains("order")){  //订单表的处理
            String[] split = line.split("\t");
            //封装outK
            outK.set(split[1]);
            //封装outV
            outV.setId(split[0]);
            outV.setPid(split[1]);
            outV.setAmount(Integer.parseInt(split[2]));
            outV.setPname("");
            outV.setFlag("order");
        }else {                             //商品表的处理
            String[] split = line.split("\t");
            //封装outK
            outK.set(split[0]);
            //封装outV
            outV.setId("");
            outV.setPid(split[0]);
            outV.setAmount(0);
            outV.setPname(split[1]);
            outV.setFlag("pd");
        }

        //写出KV
        context.write(outK,outV);
    }
}
```

（3）编写 TableReducer 类

```java
package com.atguigu.mapreduce.reducejoin;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;

public class TableReducer extends Reducer<Text,TableBean,TableBean, NullWritable> {

    @Override
    protected void reduce(Text key, Iterable<TableBean> values, Context context) throws IOException, InterruptedException {

        ArrayList<TableBean> orderBeans = new ArrayList<>();
        TableBean pdBean = new TableBean();

        for (TableBean value : values) {

            //判断数据来自哪个表
            if("order".equals(value.getFlag())){   //订单表

			  //创建一个临时TableBean对象接收value
                TableBean tmpOrderBean = new TableBean();

                try {
                    BeanUtils.copyProperties(tmpOrderBean,value);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }

			  //将临时TableBean对象添加到集合orderBeans
                orderBeans.add(tmpOrderBean);
            }else {                                    //商品表
                try {
                    BeanUtils.copyProperties(pdBean,value);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }

        //遍历集合orderBeans,替换掉每个orderBean的pid为pname,然后写出
        for (TableBean orderBean : orderBeans) {

            orderBean.setPname(pdBean.getPname());

		   //写出修改后的orderBean对象
            context.write(orderBean,NullWritable.get());
        }
    }
}
```

（4）编写 TableDriver 类

```java
package com.atguigu.mapreduce.reducejoin;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class TableDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Job job = Job.getInstance(new Configuration());

        job.setJarByClass(TableDriver.class);
        job.setMapperClass(TableMapper.class);
        job.setReducerClass(TableReducer.class);

        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(TableBean.class);

        job.setOutputKeyClass(TableBean.class);
        job.setOutputValueClass(NullWritable.class);

        FileInputFormat.setInputPaths(job, new Path("D:\\input"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\output"));

        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

**4）测试**

运行程序查看结果

```shell
1004	小米	4
1001	小米	1
1005	华为	5
1002	华为	2
1006	格力	6
1003	格力	3
```

**5）总结**

缺点：这种方式中，合并的操作是在 Reduce 阶段完成，Reduce 端的处理压力太大，Map 节点的运算负载则很低，资源利用率不高，且在 Reduce 阶段极易产生数据倾斜。

{==解决方案：Map 端实现数据合并==}。

### 3.6.3 Map Join

**1）使用场景**

Map Join 适用于一张表十分小、一张表很大的场景。

**2）优点**

!!! question "思考"

    在 Reduce 端处理过多的表，非常容易产生数据倾斜。怎么办？

在 Map 端缓存多张表，提前处理业务逻辑，这样增加 Map 端业务，减少 Reduce 端数据的压力，尽可能的减少数据倾斜。

3）具体办法：采用 DistributedCache

（1）在 Mapper 的 setup 阶段，将文件读取到缓存集合中。

（2）在 Driver 驱动类中加载缓存。

```java
//缓存普通文件到Task运行节点。
job.addCacheFile(new URI("file:///e:/cache/pd.txt"));
//如果是集群运行,需要设置HDFS路径
job.addCacheFile(new URI("hdfs://hadoop102:8020/cache/pd.txt"));
```

### 3.6.4 Map Join案例实操

**1）需求**

| id   | pid  | amount |
| ---- | ---- | ------ |
| 1001 | 01   | 1      |
| 1002 | 02   | 2      |
| 1003 | 03   | 3      |
| 1004 | 01   | 4      |
| 1005 | 02   | 5      |
| 1006 | 03   | 6      |

| pid  | pname |
| ---- | ----- |
| 01   | 小米  |
| 02   | 华为  |
| 03   | 格力  |

​	将商品信息表中数据根据商品 pid 合并到订单数据表中。

| id   | pname | amount |
| ---- | ----- | ------ |
| 1001 | 小米  | 1      |
| 1004 | 小米  | 4      |
| 1002 | 华为  | 2      |
| 1005 | 华为  | 5      |
| 1003 | 格力  | 3      |
| 1006 | 格力  | 6      |

**2）需求分析**

MapJoin 适用于关联表中有小表的情形。

![image-20230301190533360](https://cos.gump.cloud/uPic/image-20230301190533360.png)

**3）实现代码**

（1）先在 MapJoinDriver 驱动类中添加缓存文件

```java
package com.atguigu.mapreduce.mapjoin;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;

public class MapJoinDriver {

    public static void main(String[] args) throws IOException, URISyntaxException, ClassNotFoundException, InterruptedException {

        // 1 获取job信息
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        // 2 设置加载jar包路径
        job.setJarByClass(MapJoinDriver.class);
        // 3 关联mapper
        job.setMapperClass(MapJoinMapper.class);
        // 4 设置Map输出KV类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(NullWritable.class);
        // 5 设置最终输出KV类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        // 加载缓存数据
        job.addCacheFile(new URI("file:///D:/input/tablecache/pd.txt"));
        // Map端Join的逻辑不需要Reduce阶段，设置reduceTask数量为0
        job.setNumReduceTasks(0);

        // 6 设置输入输出路径
        FileInputFormat.setInputPaths(job, new Path("D:\\input"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
        // 7 提交
        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

（2）在 MapJoinMapper 类中的 setup 方法中读取缓存文件

```java
package com.atguigu.mapreduce.mapjoin;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URI;
import java.util.HashMap;
import java.util.Map;

public class MapJoinMapper extends Mapper<LongWritable, Text, Text, NullWritable> {

    private Map<String, String> pdMap = new HashMap<>();
    private Text text = new Text();

    //任务开始前将pd数据缓存进pdMap
    @Override
    protected void setup(Context context) throws IOException, InterruptedException {

        //通过缓存文件得到小表数据pd.txt
        URI[] cacheFiles = context.getCacheFiles();
        Path path = new Path(cacheFiles[0]);

        //获取文件系统对象,并开流
        FileSystem fs = FileSystem.get(context.getConfiguration());
        FSDataInputStream fis = fs.open(path);

        //通过包装流转换为reader,方便按行读取
        BufferedReader reader = new BufferedReader(new InputStreamReader(fis, "UTF-8"));

        //逐行读取，按行处理
        String line;
        while (StringUtils.isNotEmpty(line = reader.readLine())) {
            //切割一行    
//01	小米
            String[] split = line.split("\t");
            pdMap.put(split[0], split[1]);
        }

        //关流
        IOUtils.closeStream(reader);
IOUtils.closeStream(fis);
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        //读取大表数据    
//1001	01	1
        String[] fields = value.toString().split("\t");

        //通过大表每行数据的pid,去pdMap里面取出pname
        String pname = pdMap.get(fields[1]);

        //将大表每行数据的pid替换为pname
        text.set(fields[0] + "\t" + pname + "\t" + fields[2]);

        //写出
        context.write(text,NullWritable.get());
    }
}
```

## 3.7 MapReduce开发总结

**1）输入数据接口：InputFormat**

（1）默认使用的实现类是：TextInputFormat

（2）TextInputFormat 的功能逻辑是：一次读一行文本，然后将该行的起始偏移量作为 key，行内容作为 value 返回。

（3）CombineTextInputFormat 可以把多个小文件合并成一个切片处理，提高处理效率。

**2）逻辑处理接口：Mapper** 

用户根据业务需求实现其中三个方法：map()   setup()   cleanup () 

**3）Partitioner 分区**

（1）有默认实现 HashPartitioner，逻辑是根据 key 的哈希值和 numReduces 来返回一个分区号；`key.hashCode()&Integer.MAXVALUE % numReduces`

（2）如果业务上有特别的需求，可以自定义分区。

**4）Comparable 排序**

（1）当我们用自定义的对象作为 key 来输出时，就必须要实现 WritableComparable 接口，重写其中的 compareTo() 方法。

（2）部分排序：对最终输出的每一个文件进行内部排序。

（3）全排序：对所有数据进行排序，通常只有一个 Reduce。

（4）二次排序：排序的条件有两个。

**5）Combiner 合并**

Combiner 合并可以提高程序执行效率，减少 IO 传输。但是使用时必须不能影响原有的业务处理结果。

**6）逻辑处理接口：Reducer**

用户根据业务需求实现其中三个方法：reduce()   setup()   cleanup () 

**7）输出数据接口：OutputFormat**

（1）默认实现类是 TextOutputFormat，功能逻辑是：将每一个 KV 对，向目标文本文件输出一行。

（2）用户还可以自定义 OutputFormat。
