# 附录：常见错误及解决方案

## 1）连接不上MySQL数据库

（1）导错驱动包，应该把 mysql-connector-java-5.1.27-bin.jar 导入 /opt/module/hive/lib 的不是这个包。错把 mysql-connector-java-5.1.27.tar.gz 导入 hive/lib 包下。

（2）修改 user 表中的主机名称没有都修改为 %，而是修改为 localhost

## 2）Hive默认的输入格式处理是CombineHiveInputFormat，会对小文件进行合并。

```sql
set hive.input.format;

hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat
```

可以采用 HiveInputFormat 就会根据分区数输出相应的文件。

```sql
set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
```

## 3）不能执行MapReduce程序

可能是 Hadoop 的 Yarn 没开启。

## 4）启动MySQL服务时，报MySQL server PID file could not be found! 异常。

在 /var/lock/subsys/mysql 路径下创建 hadoop102.pid，并在文件中添加内容：4396

## 5）报service mysql status MySQL is not running, but lock file (/var/lock/subsys/mysql[失败])异常。

解决方案：在 /var/lib/mysql 目录下创建：-rw-rw----. 1 mysql mysql     5 12月 22 16:41 hadoop102.pid 文件，并修改权限为 777。

## 6）JVM堆内存溢出（Hive集群运行模式）

描述：java.lang.OutOfMemoryError: Java heap space

解决：在 yarn-site.xml 中加入如下代码。

```xml title="yarn-site.xml"
<property>
	<name>yarn.scheduler.maximum-allocation-mb</name>
	<value>2048</value>
</property>
<property>
  	<name>yarn.scheduler.minimum-allocation-mb</name>
  	<value>2048</value>
</property>
<property>
	<name>yarn.nodemanager.vmem-pmem-ratio</name>
	<value>2.1</value>
</property>
<property>
	<name>mapred.child.java.opts</name>
	<value>-Xmx1024m</value>
</property>
```

## 7）JVM堆内存溢出（Hive本地运行模式）

描述：在启用 Hive 本地模式后，hive.log 报错 java.lang.OutOfMemoryError: Java heap space

解决方案1（{==临时==}）：

在 Hive 客户端临时设置 io.sort.mb 和 mapreduce.task.io.sort.mb 两个参数的值为 10。

```sql
set io.sort.mb = 10;
set mapreduce.task.io.sort.mb = 10;
```

解决方案2（{==永久生效==}）：

在 $HIVE_HOME/conf 下添加 hive-env.sh。

``` sh title="hive-env.sh"
export HADOOP_HEAPSIZE=1024
```

!!! warning 

    修改之后请重启 Hive

## 8）虚拟内存限制

​	在 yarn-site.xml 中添加如下配置:

```xml title="yarn-site.xml"
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
 </property>
```

