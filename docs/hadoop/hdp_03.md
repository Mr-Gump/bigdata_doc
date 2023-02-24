# 第3章 Hadoop运行模式

1）[:link:Hadoop官方网站](http://hadoop.apache.org/)

2）Hadoop 运行模式包括：{++本地模式++}、{++伪分布式模++}式以及{++完全分布式模式++}。

- 本地模式：单机运行，只是用来演示一下官方案例。{==生产环境不用==}。
- 伪分布式模式：也是单机运行，但是具备 Hadoop 集群的所有功能，一台服务器模拟一个分布式的环境。个别缺钱的公司用来测试，生产环境不用。
- 完全分布式模式：多台服务器组成分布式环境。生产环境使用。

## 3.1 本地运行模式（官方WordCount）

1）创建在 hadoop-3.1.3 文件下面创建一个 wcinput 文件夹

```shell
mkdir wcinput
```

2）在 wcinput 文件下创建一个 word.txt 文件

```shell
cd wcinput
```

3）编辑 word.txt 文件

```shell
vim word.txt
```

在文件中输入如下内容

```txt title="word.txt"
hadoop yarn
hadoop mapreduce
atguigu
atguigu
```

保存退出：:wq

4）回到 Hadoop 目录 /opt/module/hadoop-3.1.3

5）执行程序

```shell
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount wcinput wcoutput
```

6）查看结果

```shell
cat wcoutput/part-r-00000
```

看到如下结果：

```txt title="part-r-00000"
atguigu 2
hadoop  2
mapreduce       1
yarn    1
```

## 3.2 完全分布式运行模式（开发重点）

分析：
（1）准备 3 台客户机（关闭防火墙、静态 IP、主机名称）

（2）安装 JDK

（3）配置环境变量

（4）安装 Hadoop

（5）配置环境变量

（6）配置集群

（7）单点启动

（8）配置 ssh

（9）群起并测试集群

### 3.2.1 虚拟机准备

详见 2.1、2.2 两节。

### 3.2.2 编写集群分发脚本xsync

1）scp（secure copy）安全拷贝

（1）scp 定义
scp 可以实现服务器与服务器之间的数据拷贝。（from server1 to server2）
（2）基本语法
`scp -r pdir/fname user@host:pdir/$fname`
命令   递归     要拷贝的文件路径/名称   目的地用户@主机:目的地路径/名称

（3）案例实操

{==前提==}：在 hadoop102、hadoop103、hadoop104 都已经创建好的 /opt/module、 /opt/software 两个目录，并且已经把这两个目录修改为 atguigu:atguigu

```shell
sudo chown atguigu:atguigu -R /opt/module
```

①在 hadoop102 上，将 hadoop102 中 /opt/module/jdk1.8.0_212 目录拷贝到 hadoop103 上。

```shell
scp -r /opt/module/jdk1.8.0_212  atguigu@hadoop103:/opt/module
```

②在 hadoop103 上，将 hadoop102 中 /opt/module/hadoop-3.1.3 目录拷贝到 hadoop103 上。

```shell
scp -r atguigu@hadoop102:/opt/module/hadoop-3.1.3 /opt/module/
```

③在 hadoop103 上操作，将 hadoop102 中 /opt/module 目录下所有目录拷贝到 hadoop104 上。

```shell
scp -r atguigu@hadoop102:/opt/module/* atguigu@hadoop104:/opt/module
```

2）rsync 远程同步工具

rsync 主要用于备份和镜像。具有速度快、避免复制相同内容和支持符号链接的优点。

rsync和 scp 区别：用 rsync 做文件的复制要比 scp 的速度快，rsync 只对{==差异文件==}做更新。scp 是把所有文件都复制过去。

（1）基本语法

`rsync -av $pdir/$fname $user@$host:$pdir/$fname`

命令   选项参数   要拷贝的文件路径/名称   目的地用户@主机:目的地路径/名称

  选项参数说明

| 选项 | 功能         |
| ---- | ------------ |
| -a   | 归档拷贝     |
| -v   | 显示复制过程 |

（2）案例实操

①删除 hadoop103 中 /opt/module/hadoop-3.1.3/wcinput

```shell
rm -rf wcinput/
```

②同步 hadoop102 中的 /opt/module/hadoop-3.1.3 到 hadoop103

```shell
rsync -av hadoop-3.1.3/ atguigu@hadoop103:/opt/module/hadoop-3.1.3/
```

3）xsync 集群分发脚本

（1）需求：循环复制文件到所有节点的相同目录下

（2）需求分析：

①rsync 命令原始拷贝：

```shell
rsync -av /opt/module atguigu@hadoop103:/opt/
```

②期望脚本：

xsync 要同步的文件名称

③期望脚本在任何路径都能使用（脚本放在声明了全局环境变量的路径）

```shell
echo $PATH
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/atguigu/.local/bin:/home/atguigu/bin:/opt/module/jdk1.8.0_212/bin
```

（3）脚本实现

①在 /home/atguigu/bin 目录下创建 xsync 文件

```shell
cd /home/atguigu
mkdir bin
cd bin
vim xsync
```

在该文件中编写如下代码

```sh title="xsync"
#!/bin/bash

#1. 判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi

#2. 遍历集群所有机器
for host in hadoop102 hadoop103 hadoop104
do
    echo ====================  $host  ====================
    #3. 遍历所有目录，挨个发送

    for file in $@
    do
        #4. 判断文件是否存在
        if [ -e $file ]
            then
                #5. 获取父目录
                pdir=$(cd -P $(dirname $file); pwd)

                #6. 获取当前文件的名称
                fname=$(basename $file)
                ssh $host "mkdir -p $pdir"
                rsync -av $pdir/$fname $host:$pdir
            else
                echo $file does not exists!
        fi
    done
done
```

②修改脚本 xsync 具有执行权限

```shell
chmod +x xsync
```

③测试脚本

```shell
xsync /home/atguigu/bin
```

④将脚本复制到 /bin 中，以便全局调用

```shell
sudo cp xsync /bin/
```

⑤同步环境变量配置（ root 所有者）

```shell
sudo xsync /etc/profile.d/my_env.sh
```

让环境变量生效

```shell
source /etc/profile
source /etc/profile
```

### 3.2.3 SSH无密登录配置

**1）配置 ssh**

（1）基本语法

ssh 另一台电脑的 IP 地址

（2）ssh 连接时出现 Host key verification failed 的解决方法

```shell
ssh hadoop103
```

如果出现如下内容

```shell
Are you sure you want to continue connecting (yes/no)? 
```

输入 yes，并回车

（3）退回到 hadoop102

```shell
exit
```

**2）无密钥配置**

（1）免密登录原理 

![image-20230224181737063](https://cos.gump.cloud/uPic/image-20230224181737063.png)

（2）生成公钥和私钥

```shell
ssh-keygen -t rsa
```

然后敲（三个回车），就会生成两个文件 id_rsa（私钥）、id_rsa.pub（公钥）

（3）将公钥拷贝到要免密登录的目标机器上

```shell
ssh-copy-id hadoop102
ssh-copy-id hadoop103
ssh-copy-id hadoop104
```

!!! info "注意"
    还需要在 hadoop103 上采用 atguigu 账号配置一下无密登录到 hadoop102、hadoop103、hadoop104 服务器上。
    还需要在 hadoop104 上采用 atguigu 账号配置一下无密登录到 hadoop102、hadoop103、hadoop104 服务器上。
    还需要在 hadoop102 上采用 root 账号，配置一下无密登录到 hadoop102、hadoop103、hadoop104；
   **3）.ssh 文件夹下（~/.ssh）的文件功能解释**

| known_hosts     | 记录ssh访问过计算机的公钥（public key） |
| --------------- | --------------------------------------- |
| id_rsa          | 生成的私钥                              |
| id_rsa.pub      | 生成的公钥                              |
| authorized_keys | 存放授权过的无密登录服务器公钥          |

### 3.2.4 集群配置

**1）集群部署规划**

!!! warning "注意"
    NameNode 和 SecondaryNameNode 不要安装在同一台服务器
    ResourceManager 也很消耗内存，不要和 NameNode、SecondaryNameNode 配置在同一台机器上。

|      | hadoop102        | hadoop103                  | hadoop104                 |
| ---- | ---------------- | -------------------------- | ------------------------- |
| HDFS | NameNodeDataNode | DataNode                   | SecondaryNameNodeDataNode |
| YARN | NodeManager      | ResourceManagerNodeManager | NodeManager               |

**2）配置文件说明**

Hadoop 配置文件分两类：默认配置文件和自定义配置文件，只有用户想修改某一默认配置值时，才需要修改自定义配置文件，更改相应属性值。

（1）默认配置文件：

| 要获取的默认文件     | 文件存放在 Hadoop 的 jar 包中的位置                       |
| -------------------- | --------------------------------------------------------- |
| [core-default.xml]   | hadoop-common-3.1.3.jar/core-default.xml                  |
| [hdfs-default.xml]   | hadoop-hdfs-3.1.3.jar/hdfs-default.xml                    |
| [yarn-default.xml]   | hadoop-yarn-common-3.1.3.jar/yarn-default.xml             |
| [mapred-default.xml] | hadoop-mapreduce-client-core-3.1.3.jar/mapred-default.xml |

（2）自定义配置文件：

​	core-site.xml、hdfs-site.xml、yarn-site.xml、mapred-site.xml 四个配置文件存放在 $HADOOP_HOME/etc/hadoop 这个路径上，用户可以根据项目需求重新进行修改配置。

**3）配置集群**

（1）核心配置文件

配置 core-site.xml

```xml title="core-site.xml"
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- 指定NameNode的地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop102:8020</value>
    </property>

    <!-- 指定hadoop数据的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-3.1.3/data</value>
    </property>

    <!-- 配置HDFS网页登录使用的静态用户为atguigu -->
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>atguigu</value>
    </property>
</configuration>
```

（2）HDFS 配置文件

```xml title="hdfs-site.xml"
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- nn web端访问地址-->
	<property>
        <name>dfs.namenode.http-address</name>
        <value>hadoop102:9870</value>
    </property>
	<!-- 2nn web端访问地址-->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop104:9868</value>
    </property>
</configuration>
```

（3）YARN 配置文件

```xml title="yarn-site.xml"
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- 指定MR走shuffle -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!-- 指定ResourceManager的地址-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop103</value>
    </property>

    <!-- 环境变量的继承 -->
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

（4）MapReduce 配置文件

```xml title="mapred-site.xml"
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<!-- 指定MapReduce程序运行在Yarn上 -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

**4）在集群上分发配置好的 Hadoop 配置文件**

```shell
xsync /opt/module/hadoop-3.1.3/etc/hadoop/
```

**5）去 103 和 104 上查看文件分发情况**

```shel
cat /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml
cat /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml
```

### 3.2.5 群起集群

**1）配置 workers**

```shell title="workers"
hadoop102
hadoop103
hadoop104
```

!!! warning "注意"

    该文件中添加的内容结尾不允许有空格，文件中不允许有空行。

同步所有节点配置文件

```shell
xsync /opt/module/hadoop-3.1.3/etc
```

**2）启动集群**

（1）如果集群是第一次启动，需要在 hadoop102 节点格式化 NameNode（注意：格式化 NameNode，会产生新的集群 id，导致 NameNode 和 DataNode 的集群 id 不一致，集群找不到已往数据。如果集群在运行过程中报错，需要重新格式化 NameNode 的话，一定要先停止 namenode 和 datanode 进程，并且要删除所有机器的 data 和 logs 目录，然后再进行格式化。）

``` shell
hdfs namenode -format
```

（2）启动 HDFS

```shell 
 sbin/start-dfs.sh
```

（3）在配置了 ResourceManager 的节点（hadoop103）启动 YARN

```shell 
sbin/start-yarn.sh
```

（4）Web 端查看 HDFS 的 NameNode

①浏览器中输入：http://hadoop102:9870

②查看 HDFS 上存储的数据信息

（5）Web 端查看 YARN 的 ResourceManager

①浏览器中输入：http://hadoop103:8088

②查看 YARN 上运行的 Job 信息

**3）集群基本测试**

（1）上传文件到集群

上传小文件

```shell
hadoop fs -mkdir /input
hadoop fs -put $HADOOP_HOME/wcinput/word.txt /input
```

上传大文件

```shell
hadoop fs -put  /opt/software/jdk-8u212-linux-x64.tar.gz  /
```

（2）上传文件后查看文件存放在什么位置

查看 HDFS 文件存储路径

```shell
pwd

/opt/module/hadoop-3.1.3/data/dfs/data/current/BP-1436128598-192.168.10.102-1610603650062/current/finalized/subdir0/subdir0
```

查看 HDFS 在磁盘存储文件内容

```shell
cat blk_1073741825

hadoop yarn
hadoop mapreduce 
atguigu
atguigu
```

（3）拼接

```shell
-rw-rw-r--. 1 atguigu atguigu 134217728 5月  23 16:01 blk_1073741836
-rw-rw-r--. 1 atguigu atguigu   1048583 5月  23 16:01 blk_1073741836_1012.meta
-rw-rw-r--. 1 atguigu atguigu  63439959 5月  23 16:01 blk_1073741837
-rw-rw-r--. 1 atguigu atguigu    495635 5月  23 16:01 blk_1073741837_1013.meta
```

```shell
cat blk_1073741836>>tmp.tar.gz
cat blk_1073741837>>tmp.tar.gz
tar -zxvf tmp.tar.gz
```

（4）下载

```shell
hadoop fs -get /jdk-8u212-linux-x64.tar.gz ./
```

（5）执行 wordcount 程序

```shell
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /input /output
```

### 3.2.6 配置历史服务器

为了查看程序的历史运行情况，需要配置一下历史服务器。具体配置步骤如下：

**1）配置 mapred-site.xml**

```xml title="mapped-site.xml"
<!-- 历史服务器端地址 -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoop102:10020</value>
</property>

<!-- 历史服务器web端地址 -->
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop102:19888</value>
</property>
```

**2）分发配置**

```shell
xsync $HADOOP_HOME/etc/hadoop/mapred-site.xml
```

**3）在 hadoop102 启动历史服务器**

```shell
mapred --daemon start historyserver
```

**4）查看历史服务器是否启动**

```shell
jps
```

**5）查看 JobHistory**

http://hadoop102:19888/jobhistory

### 3.2.7 配置日志的聚集

日志聚集概念：应用运行完成以后，将程序运行日志信息上传到 HDFS 系统上。

![image-20230224183339687](https://cos.gump.cloud/uPic/image-20230224183339687.png)

日志聚集功能好处：可以方便的查看到程序运行详情，方便开发调试。

!!! info "注意"

    开启日志聚集功能，需要重新启动 NodeManager 、ResourceManager 和 HistoryServer。

开启日志聚集功能具体步骤如下：

**1）配置 yarn-site.xml**

```xml title="yarn-site.xml"
<!-- 开启日志聚集功能 -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>
<!-- 设置日志聚集服务器地址 -->
<property>  
    <name>yarn.log.server.url</name>  
    <value>http://hadoop102:19888/jobhistory/logs</value>
</property>
<!-- 设置日志保留时间为7天 -->
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
</property>
```

**2）分发配置**

```shell
xsync $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

**3）关闭 NodeManager 、ResourceManager 和 HistoryServer**

```shell
sbin/stop-yarn.sh
mapred --daemon stop historyserver
```

**4）启动 NodeManager 、ResourceManage 和 HistoryServer**

```shell
start-yarn.sh
mapred --daemon start historyserver
```

**5）删除 HDFS 上已经存在的输出文件**

```shell
hadoop fs -rm -r /output
```

**6）执行 WordCount 程序**

```shell
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /input /output
```

**7）查看日志**

（1）历史服务器地址

http://hadoop102:19888/jobhistory

（2）历史任务列表

![image-20230224183616800](https://cos.gump.cloud/uPic/image-20230224183616800.png)

（3）查看任务运行日志

![image-20230224183626351](https://cos.gump.cloud/uPic/image-20230224183626351.png)

（4）运行日志详情

![image-20230224183642098](https://cos.gump.cloud/uPic/image-20230224183642098.png)

### 3.2.8 集群启动/停止方式总结

**1）各个模块分开启动/停止（配置 ssh 是前提）**

（1）整体启动/停止 HDFS

```shell
start-dfs.sh/stop-dfs.sh
```

（2）整体启动/停止 YARN

```shell
start-yarn.sh/stop-yarn.sh
```

**2）各个服务组件逐一启动/停止**

（1）分别启动/停止 HDFS 组件

```shell
hdfs --daemon start/stop namenode/datanode/secondarynamenode
```

（2）启动/停止 YARN

```shell
yarn --daemon start/stop resourcemanager/nodemanager
```

### 3.2.9 编写Hadoop集群常用脚本

**1）Hadoop 集群启停脚本（包含 HDFS，Yarn，Historyserver）：myhadoop.sh**

```shell
cd /home/atguigu/bin
vim myhadoop.sh
```

输入如下内容

```sh title="myhadoop.sh"
#!/bin/bash

if [ $# -lt 1 ]
then
    echo "No Args Input..."
    exit ;
fi

case $1 in
"start")
        echo " =================== 启动 hadoop集群 ==================="

        echo " --------------- 启动 hdfs ---------------"
        ssh hadoop102 "/opt/module/hadoop-3.1.3/sbin/start-dfs.sh"
        echo " --------------- 启动 yarn ---------------"
        ssh hadoop103 "/opt/module/hadoop-3.1.3/sbin/start-yarn.sh"
        echo " --------------- 启动 historyserver ---------------"
        ssh hadoop102 "/opt/module/hadoop-3.1.3/bin/mapred --daemon start historyserver"
;;
"stop")
        echo " =================== 关闭 hadoop集群 ==================="

        echo " --------------- 关闭 historyserver ---------------"
        ssh hadoop102 "/opt/module/hadoop-3.1.3/bin/mapred --daemon stop historyserver"
        echo " --------------- 关闭 yarn ---------------"
        ssh hadoop103 "/opt/module/hadoop-3.1.3/sbin/stop-yarn.sh"
        echo " --------------- 关闭 hdfs ---------------"
        ssh hadoop102 "/opt/module/hadoop-3.1.3/sbin/stop-dfs.sh"
;;
*)
    echo "Input Args Error..."
;;
esac
```

保存后退出，然后赋予脚本执行权限

```shell
chmod +x myhadoop.sh
```

**2）查看三台服务器 Java 进程脚本：jpsall**

```shell
cd /home/atguigu/bin
vim jpsall
```

输入如下内容

```shell title="jps"
#!/bin/bash

for host in hadoop102 hadoop103 hadoop104
do
        echo =============== $host ===============
        ssh $host jps 
done
```

保存后退出，然后赋予脚本执行权限

```shell
chmod +x jpsall
```

**3）分发 /home/atguigu/bin 目录，保证自定义脚本在三台机器上都可以使用**

```shell
xsync /home/atguigu/bin/
```

### 3.2.10 常用端口号说明

| 端口名称                  | Hadoop2.x   | Hadoop3.x        |
| ------------------------- | ----------- | ---------------- |
| NameNode内部通信端口      | 8020 / 9000 | 8020 / 9000/9820 |
| NameNode HTTP UI          | 50070       | 9870             |
| MapReduce查看执行任务端口 | 8088        | 8088             |
| 历史服务器通信端口        | 19888       | 19888            |

### 3.2.11 集群时间同步（了解...）

{==如果服务器在公网环境（能连接外网），可以不采用集群时间同步==}，因为服务器会定期和公网时间进行校准；

如果服务器在内网环境，必须要配置集群时间同步，否则时间久了，会产生时间偏差，导致集群执行任务时间不同步。

**1）需求**

找一个机器，作为时间服务器，所有的机器与这台集群时间进行定时的同步，生产环境根据任务对时间的准确程度要求周期同步。测试环境为了尽快看到效果，采用 1 分钟同步一次。

![image-20230224184120320](https://cos.gump.cloud/uPic/image-20230224184120320.png)

**2）时间服务器配置（必须 root 用户）**

（1）查看{==所有节点==} ntpd 服务状态和开机自启动状态

```shell
sudo systemctl status ntpd
sudo systemctl start ntpd
sudo systemctl is-enabled ntpd
```

（2）修改 hadoop102 的 ntp.conf 配置文件

```shell
sudo vim /etc/ntp.conf
```

修改内容如下

①修改 1（授权 192.168.10.0 - 192.168.10.255 网段上的所有机器可以从这台机器上查询和同步时间）

```conf title="ntp.conf"
#restrict 192.168.10.0 mask 255.255.255.0 nomodify notrap
restrict 192.168.10.0 mask 255.255.255.0 nomodify notrap
```

②修改 2（集群在局域网中，不使用其他互联网上的时间）

```conf title="ntp.conf"
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
```

③添加 3（当该节点丢失网络连接，依然可以采用本地时间作为时间服务器为集群中的其他节点提供时间同步）

```conf title="ntp.conf"
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```

（3）修改 hadoop102 的 /etc/sysconfig/ntpd 文件

增加内容如下（让硬件时间与系统时间一起同步）

```conf title="ntp.conf"
SYNC_HWCLOCK=yes
```

（4）重新启动 ntpd 服务

```shell
sudo systemctl start ntpd
```

（5）设置 ntpd 服务开机启动

```shell
sudo systemctl enable ntpd
```

**3）其他机器配置（必须 root 用户）**

（1）关闭{==所有节点==}上 ntp 服务和自启动

```shell
sudo systemctl stop ntpd
sudo systemctl disable ntpd
```

（2）在其他机器配置 1 分钟与时间服务器同步一次

```shell
sudo crontab -e
```

编写定时任务如下：

```shell
*/1 * * * * /usr/sbin/ntpdate hadoop102
```

（3）修改任意机器时间

```shell
sudo date -s "2021-9-11 11:11:11"
```

（4）1 分钟后查看机器是否与时间服务器同步

```shell
sudo date
```

