# 第2章 Hive on Tez部署

## 2.1 Hadoop集群配置

1）上传 tez 依赖到 HDFS

```bash
hadoop fs -mkdir /tez
hadoop fs -put /opt/software/tez-0.10.1.tar.gz /tez
```

2）新建 tez-site.xml

```bash
vim $HADOOP_HOME/etc/hadoop/tez-site.xml
```

```xml title="tea-site.xml"
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>tez.lib.uris</name>
        <value>${fs.defaultFS}/tez/tez-0.10.1.tar.gz</value>
    </property>
    <property>
        <name>tez.use.cluster.hadoop-libs</name>
        <value>true</value>
    </property>
    <property>
        <name>tez.am.resource.memory.mb</name>
        <value>1024</value>
    </property>
    <property>
        <name>tez.am.resource.cpu.vcores</name>
        <value>1</value>
    </property>
</configuration>
```

3）分发 tez-site.xml 文件

```bash
xsync tez-site.xml
```

## 2.2 客户端节点（Hive所在节点）配置

1）将 tez 最小版安装包拷贝到 hadoop102，并解压 tar 包

```bash
mkdir /opt/module/tez
tar -zxvf /opt/software/tez-0.10.1-minimal.tar.gz -C /opt/module/tez
```

2）将 tez 增加到 Hadoop Classpath 中

修改 Hadoop 环境变量配置文件

```bash
vim $HADOOP_HOME/etc/hadoop/shellprofile.d/tez.sh
```

添加 Tez 的 Jar 包相关信息

```bash
hadoop_add_profile tez
function _tez_hadoop_classpath
{
hadoop_add_classpath "$HADOOP_HOME/etc/hadoop" after
hadoop_add_classpath "/opt/module/tez/*" after
hadoop_add_classpath "/opt/module/tez/lib/*" after
}
```

3）修改 Hive 的计算引擎

```bash
vim $HIVE_HOME/conf/hive-site.xml
```

```xml title="hive-site.xml"
<property>
    <name>hive.execution.engine</name>
    <value>tez</value>
</property>
<property>
    <name>hive.tez.container.size</name>
    <value>1024</value>
</property>
```

## 2.3 Hive on Tez测试

1）启动 hive 客户端

```bash
bin/hive
```

2）创建一张测试表

```sql
create table student(id int, name string);
```

3）通过 insert 测试效果

```bash
insert into table student values(1,'abc');
```

若结果如下，则说明配置成功

![image-20230206222524682](https://cos.gump.cloud/uPic/image-20230206222524682.png)

## 2.4 Tez UI部署

Tez UI 提供了一个 Web 页面用于展示 Tez 应用程序的详细信息，供开发人员调试使用。

### 2.4.1 部署Yarn Timeline Server

Yarn Timeline Server 可用于持久化存储 Yarn 中运行的给种类型的任务（MR、Spark、Tez 等）信息。并且，它提供了丰富的 REST API 来查询这些应用的信息。

Tez UI 就是基于 Yarn Timeline Server 的一个纯前端应用，用于展示 Yarn Timeline Server 中保存的 Tez 应用信息。

1）配置 Yarn Timeline Server

（1）修改 $HADOOP_HOME/etc/yarn-site.xml 文件，并分发

```xml title="yarn-site.xml"
<property>
    <name>yarn.timeline-service.enabled</name>
    <value>true</value>
</property>
<property>
    <name>yarn.timeline-service.hostname</name>
    <value>hadoop102</value>
</property>
<property>
    <name>yarn.timeline-service.http-cross-origin.enabled</name>
    <value>true</value>
</property>
<property>
  <name>yarn.resourcemanager.system-metrics-publisher.enabled</name>
    <value>true</value>
</property>
```

（2）重启 Yarn 集群
（3）启动 Yarn Timeline Server

```bash
yarn --daemon start timelineserver
```

（4）访问 Yarn Timeline Server 的 Web UI

地址为：http://hadoop102:8188 

### 2.4.2 部署Tez UI

1）部署 Tomcat

前文提到，Tez UI 是一个纯前端应用，故将其部署到任意一个 Web 容器即可。本文选择使用 Tomcat。

（1）解压 tomcat 安装包

```bash
tar -zxvf apache-tomcat-9.0.68.tar.gz -C /opt/module/
mv /opt/module/apache-tomcat-9.0.68/ /opt/module/tomcat
```

（2）配置环境变量

修改 /etc/profile.d/my_env.sh 配置文件，增加以下内容

```sh title="my_env.sh"
#TOMCAT_HOME
export TOMCAT_HOME=/opt/module/tomcat
export PATH=$PATH:$TOMCAT_HOME/bin
```

重新加载配置文件

```shell
source /etc/profile.d/my_env.sh
```

2）部署 Tez UI

（1）清空 $TOMCAT_HOME/webapps 目录

```shell
rm -rf /opt/module/tomcat/webapps/*
```

（2）部署 tez-ui-0.10.1.war 到 tomcat

```shell
mkdir /opt/module/tomcat/webapps/tez-ui

unzip tez-ui-0.10.1.war -d /opt/module/tomcat/webapps/tez-ui
```

注：webapps 下的目录会自动映射到访问地址，也就是将来 tez-ui 的访问地址为（ 8080 为 tomcat 的默认端口号）http://hadoop102:8080/tez-ui

（3）修改 tez ui 配置文件

```shell
vim /opt/module/tomcat/webapps/tez-ui/config/configs.js
```

```javascript title="configs.js"
timeline: "http://hadoop102:8188",
rm: "http://hadoop103:8088",
```

3）配置 tez-site.xml 文件

（1）修改 $HADOOP_HOME/etc/hadoop/tez-site.xml 文件，增加如下内容

```xml title="tez-site.xml"
<property>
    <name>tez.history.logging.service.class</name>
    <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
</property>
<property>
    <name>tez.tez-ui.history-url.base</name>
    <value>http://hadoop102:8080/tez-ui</value>
</property>
```

（2）分发 tez-site.xml 文件
（3）重启 Yarn

4）配置 Hive Hook

修改 hive-site.xml 配置文件，增加如下内容

```xml title="hive-site.xml"
<property>
    <name>hive.exec.pre.hooks</name>
    <value>org.apache.hadoop.hive.ql.hooks.ATSHook</value>
</property>
<property>
    <name>hive.exec.post.hooks</name>
    <value>org.apache.hadoop.hive.ql.hooks.ATSHook</value>
</property>
<property>
    <name>hive.exec.failure.hooks</name>
    <value>org.apache.hadoop.hive.ql.hooks.ATSHook</value>
</property>
```

5）启动 Tomcat

```shell
startup.sh
```

6）访问 tez ui

地址如下：http://hadoop102:8080/tez-ui

ui 界面如下：

![image-20230206223602536](https://cos.gump.cloud/uPic/image-20230206223602536.png)

