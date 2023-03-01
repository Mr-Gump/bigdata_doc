# 第7章 常见错误及解决方案

1）导包容易出错。尤其 Text 和 CombineTextInputFormat。

2）Mapper 中第一个输入的参数必须是 LongWritable 或者 NullWritable，不可以是 IntWritable.  报的错误是类型转换异常。

3）`java.lang.Exception: java.io.IOException: Illegal partition for 13926435656 (4)`，说明 Partition 和 ReduceTask 个数没对上，调整 ReduceTask 个数。

4）如果分区数不是 1，但是 reducetask 为 1，是否执行分区过程。答案是：不执行分区过程。因为在 MapTask 的源码中，执行分区的前提是先判断 ReduceNum 个数是否大于 1。不大于 1 肯定不执行。

5）在 Windows 环境编译的 jar 包导入到 Linux 环境中运行，

`hadoop jar wc.jar com.atguigu.mapreduce.wordcount.WordCountDriver /user/atguigu/ /user/atguigu/output`

报如下错误：

```shell
Exception in thread "main" java.lang.UnsupportedClassVersionError: com/atguigu/mapreduce/wordcount/WordCountDriver : Unsupported major.minor version 52.0
```

原因是 Windows 环境用的 jdk1.7，Linux 环境用的 jdk1.8。

解决方案：统一 jdk 版本。

6）缓存 pd.txt 小文件案例中，报找不到 pd.txt 文件

原因：大部分为路径书写错误。还有就是要检查 pd.txt 的问题。还有个别电脑写相对路径找不到 pd.txt，可以修改为绝对路径。

7）报类型转换异常。

通常都是在驱动函数中设置 Map 输出和最终输出时编写错误。

Map 输出的 key 如果没有排序，也会报类型转换异常。

8）集群中运行 wc.jar 时出现了无法获得输入文件。

原因：WordCount 案例的输入文件不能放用 HDFS 集群的根目录。

9）出现了如下相关异常

```shell
Exception in thread "main" java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Native Method)
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access(NativeIO.java:609)
	at org.apache.hadoop.fs.FileUtil.canRead(FileUtil.java:977)
java.io.IOException: Could not locate executable null\bin\winutils.exe in the Hadoop binaries.
	at org.apache.hadoop.util.Shell.getQualifiedBinPath(Shell.java:356)
	at org.apache.hadoop.util.Shell.getWinUtilsPath(Shell.java:371)
	at org.apache.hadoop.util.Shell.<clinit>(Shell.java:364)
```

解决方案：拷贝 hadoop.dll 文件到 Windows 目录 C:\Windows\System32。个别同学电脑还需要修改 Hadoop 源码。

方案二：创建如下包名，并将 NativeIO.java 拷贝到该包名下

10）自定义 Outputformat 时，注意在 RecordWirter 中的 close 方法必须关闭流资源。否则输出的文件内容中数据为空。

```java
@Override
public void close(TaskAttemptContext context) throws IOException, InterruptedException {
		if (atguigufos != null) {
			atguigufos.close();
		}
		if (otherfos != null) {
			otherfos.close();
		}
}
```