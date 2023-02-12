# 第2章 Hive安装

## 2.1 Hive安装地址

1）[:material-link:Hive 官网](http://hive.apache.org/)

2）[:material-file-document:文档](https://cwiki.apache.org/confluence/display/Hive/GettingStarted)

3）[:material-download:下载](http://archive.apache.org/dist/hive/)

4）[:material-github:Github 仓库](https://github.com/apache/hive)

## 2.2 Hive安装部署

### 2.2.1 安装Hive

1）把 apache-hive-3.1.3-bin.tar.gz 上传到 Linux 的 /opt/software 目录下

2）解压 apache-hive-3.1.3-bin.tar.gz 到 /opt/module/ 目录下面

```shell
tar -zxvf /opt/software/apache-hive-3.1.3-bin.tar.gz -C /opt/module/
```

3）修改 apache-hive-3.1.3-bin.tar.gz 的名称为 hive

```shell
mv /opt/module/apache-hive-3.1.3-bin/ /opt/module/hive
```

4）修改 /etc/profile.d/my_env.sh，添加环境变量

```shell
sudo vim /etc/profile.d/my_env.sh
```

（1）添加内容

```sh title="env.sh"
#HIVE_HOME
export HIVE_HOME=/opt/module/hive
export PATH=$PATH:$HIVE_HOME/bin
```

（2）source 一下

```shell
source /etc/profile.d/my_env.sh
```

5）初始化元数据库（默认是 derby 数据库）

```shell
bin/schematool -dbType derby -initSchema
```

### 2.2.2 启动并使用Hive

1）启动 Hive

```shell
bin/hive
```

2）使用 Hive

```sql
show databases;
show tables;
create table stu(id int, name string);
insert into stu values(1,"ss");
select * from stu;
```

观察 HDFS 的路径 /user/hive/warehouse/stu，体会 Hive 与 Hadoop 之间的关系。

{==Hive 中的表在 Hadoop 中是目录；Hive 中的数据在 Hadoop 中是文件==}。

![image-20230212213616356](https://cos.gump.cloud/uPic/image-20230212213616356.png)

3）在 Xshell 窗口中开启另一个窗口开启 Hive，在 /tmp/atguigu 目录下监控 hive.log 文件

<div class="termy">
```console
$ tail -f hive.log


Caused by: ERROR XSDB6: Another instance of Derby may have already booted the database /opt/module/hive/metastore_db.
        at org.apache.derby.iapi.error.StandardException.newException(Unknown Source)
        at org.apache.derby.iapi.error.StandardException.newException(Unknown Source)
        at org.apache.derby.impl.store.raw.data.BaseDataFileFactory.privGetJBMSLockOnDB(Unknown Source)
        at org.apache.derby.impl.store.raw.data.BaseDataFileFactory.run(Unknown Source)
...
```
</div>

!!! info

    原因在于 Hive 默认使用的元数据库为 derby。{==derby 数据库的特点是同一时间只允许一个客户端访问==}。如果多个 Hive 客户端同时访问，就会报错。由于在企业开发中，都是多人协作开发，需要多客户端同时访问 Hive，怎么解决呢？我们可以将 Hive 的元数据改为用 MySQL 存储，MySQL 支持多客户端同时访问。

![image-20230212214112596](https://cos.gump.cloud/uPic/image-20230212214112596.png)

4）首先退出 Hive 客户端。然后在 Hive 的安装目录下将 derby.log 和 metastore_db 删除，顺便将 HDFS 上目录删除

```shell
rm -rf derby.log metastore_db
hadoop fs -rm -r /user
```

5）删除 HDFS 中 /user/hive/warehouse/stu 中数据

![image-20230212214201486](https://cos.gump.cloud/uPic/image-20230212214201486.png)

![image-20230212214206198](https://cos.gump.cloud/uPic/image-20230212214206198.png)

## 2.3 MySQL安装

### 2.3.1 安装准备

1）如果是虚拟机服务器，按照如下步骤准备 MySQL 安装前环境

（1）卸载自带的 mariadb 以及如果之前安装过 MySQL，需要全都卸载掉

①先查看是否安装过 MySQL 或者 mariadb

```shell
rpm -qa | grep -i -E mysql\|mariadb
```

②出现以上任何一个都需要卸载（之前安装 MYSQL 版本不同内容也可能不同）

```shell
rpm -qa | grep -i -E mysql\|mariadb | xargs -n1 sudo rpm -e --nodeps
```

（2）如果之前安装过 MySQL 需要清空原先的所有数据

①通过 /etc/my.cnf 查看 MySQL 数据的存储位置

```shell
sudo cat /etc/my.cnf
```

②去往 /var/lib/mysql 以下得拿 root 权限

```shell
su - root
cd /var/lib/mysql
rm -rf * 
```

2）如果是阿里云服务器，按照如下步骤准备 MySQL 安装前环境

!!! info "说明"

    由于阿里云服务器安装的是 Linux 最小系统版，没有如下工具，所以需要安装。

（1）卸载 MySQL 依赖，虽然机器上没有装 MySQL，但是这一步{==不可少==}

```shell
sudo yum remove mysql-libs
```

（2）下载依赖并安装

```shell
sudo yum install libaio
sudo yum -y install autoconf
```

### 2.3.2 安装MySQL

1）上传 MySQL 安装包以及 MySQL 驱动jar包

```shell
mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar
mysql-connector-java-5.1.37.jar
```

2）解压 MySQL 安装包

```shell
mkdir mysql_lib
tar -xf mysql-5.7.28-1.el7.x86_64.rpm-bundle.tar -C mysql_lib/
```

3）安装 MySQL 依赖

```shell
cd mysql_lib
sudo rpm -ivh mysql-community-common-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-libs-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-libs-compat-5.7.28-1.el7.x86_64.rpm
```

4）安装 mysql-client

```shell
sudo rpm -ivh mysql-community-client-5.7.28-1.el7.x86_64.rpm
```

5）安装 mysql-server

```shell
sudo rpm -ivh mysql-community-server-5.7.28-1.el7.x86_64.rpm
```

!!! warning "注意"

    如果报如下错误，这是由于 yum 安装了旧版本的 GPG keys 所造成，从 rpm 版本 4.1 后，在安装或升级软件包时会自动检查软件包的签名
    
    ```shell
    warning: 05_mysql-community-server-5.7.16-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
    error: Failed dependencies:
    libaio.so.1()(64bit) is needed by mysql-community-server-5.7.16-1.el7.x86_64
    ```
    
    解决办法
    
    ```shell
    sudo yum -y install libaio
    ```

6）启动 MySQL

```shell
sudo systemctl start mysqld
```

7）查看 MySQL 密码

```shell
sudo cat /var/log/mysqld.log | grep password
```

### 2.3.3 配置MySQL

配置只要是 root 用户 + 密码，在任何主机上都能登录 MySQL 数据库。

1）用刚刚查到的密码进入 MySQL（如果报错，给密码加单引号）

```shell
mysql -uroot -p'password'
```

2）设置复杂密码（由于 MySQL 密码策略，此密码必须足够复杂）

```sql
set password=password("Qs23=zs32");
```

3）更改 MySQL 密码策略

```sql
set global validate_password_length=4;
set global validate_password_policy=0;
```

4）设置简单好记的密码

```sql
set password=password("123456");
```

5）进入 MySQL 库

```sql
use mysql
```

6）查询 user 表

```sql
select user, host from user;
```

7）修改 user 表，把 Host 表内容修改为 %

```sql
update user set host="%" where user="root";
```

8）刷新

```sql
flush privileges;
```

9）退出

```sql
quit;
```

## 2.4 配置Hive元数据存储到MySQL

![image-20230212222119071](https://cos.gump.cloud/uPic/image-20230212222119071.png)

### 2.4.1 配置元数据到MySQL

1）将 MySQL 的 JDBC 驱动拷贝到 Hive 的 lib 目录下。

```shell
cp /opt/software/mysql-connector-java-5.1.37.jar $HIVE_HOME/lib
```

2）在 $HIVE_HOME/conf 目录下新建 hive-site.xml 文件

```shell
vim $HIVE_HOME/conf/hive-site.xml
```

```xml title="hive-site.xml"
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- jdbc连接的URL -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
    </property>
    
    <!-- jdbc连接的Driver-->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    
	<!-- jdbc连接的username-->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <!-- jdbc连接的password -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>

    <!-- Hive默认在HDFS的工作目录 -->
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
</configuration>
```

3）登录 MySQL

```shell
mysql -uroot -p123456
```

4）新建 Hive 元数据库

```sql
create database metastore;
quit;
```

5）初始化 Hive 元数据库（修改为采用 MySQL 存储元数据）

```shell
bin/schematool -dbType mysql -initSchema -verbose
```

### 2.4.2 验证元数据是否配置成功

1）再次启动 Hive

```shell
bin/hive
```

2）使用 Hive

```sql
show databases;
show tables;
create table stu(id int, name string);
insert into stu values(1,"ss");
select * from stu;
```

3）在 Xshell 窗口中开启另一个窗口开启 Hive（两个窗口都可以操作 Hive，没有出现异常）

```sql
show databases;
show tables;
select * from stu;
```

### 2.4.3 查看MySQL中的元数据

1）登录 MySQL

```shell
mysql -uroot -p123456
```

2）查看元数据库 metastore

```sql
show databases;
use metastore;
show tables;
```

（1）查看元数据库中存储的{==库信息==}

<div class="termy">
```console
$ select * from DBS;

+-------+-----------------------+-------------------------------------------+---------+------------+------------+-----------+
| DB_ID | DESC                  | DB_LOCATION_URI                           | NAME    | OWNER_NAME | OWNER_TYPE | CTLG_NAME |
+-------+-----------------------+-------------------------------------------+---------+------------+------------+-----------+
|     1 | Default Hive database | hdfs://hadoop102:8020/user/hive/warehouse | default | public     | ROLE       | hive      |
+-------+-----------------------+-------------------------------------------+---------+------------+------------+-----------+
```
</div>


（2）查看元数据库中存储的{==表信息==}

<div class="termy">
```console
$ select * from TBLS;

+--------+-------------+-------+------------------+---------+------------+-----------+-------+----------+---------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER   | OWNER_TYPE | RETENTION | SD_ID | TBL_NAME | TBL_TYPE      | 
+--------+-------------+-------+------------------+---------+------------+-----------+-------+----------+---------------+
|      1 |  1656318303 |     1 |                0 | atguigu | USER       |         0 |     1 | stu      | MANAGED_TABLE |
+--------+-------------+-------+------------------+---------+------------+-----------+-------+----------+---------------+
```
</div>


（3）查看元数据库中存储的表中{==列==}相关信息

<div class="termy">
```conosle
$ select * from TAB_COL_STATS;

+-------+----------+---------+------------+-------------+-------------+--------+
| CS_ID | CAT_NAME | DB_NAME | TABLE_NAME | COLUMN_NAME | COLUMN_TYPE | TBL_ID |
+-------+----------+---------+------------+-------------+-------------+--------+
|     1 | hive     | default | stu        | id          | int         |      1 |
|     2 | hive     | default | stu        | name        | string      |      1 |
+-------+----------+---------+------------+-------------+-------------+--------+
```
</div>

（4）查看元数据库中数据P==存储位置==}和{==读入写出==}的方式

```

```sql
select * from SDS;
```

![image-20230212222823139](https://cos.gump.cloud/uPic/image-20230212222823139.png)

## 2.5 Hive服务部署

### 2.5.1 hiveserver2服务

Hive 的 hiveserver2 服务的作用是提供 jdbc / odbc 接口，为用户提供远程访问 Hive 数据的功能，例如用户期望在个人电脑中访问远程服务中的 Hive 数据，就需要用到 Hiveserver2。

![image-20230212222913720](https://cos.gump.cloud/uPic/image-20230212222913720.png)

1）用户说明

!!! question 

    在远程访问 Hive 数据时，客户端并未直接访问 Hadoop 集群，而是由 Hivesever2 代理访问。由于 Hadoop 集群中的数据具备访问权限控制，所以此时需考虑一个问题：那就是访问 Hadoop 集群的用户身份是谁？是 Hiveserver2 的启动用户？还是客户端的登录用户？

答案是都有可能，具体是谁，由 Hiveserver2 的 hive.server2.enable.doAs 参数决定，该参数的含义是是否启用 Hiveserver2 用户模拟的功能。若启用，则 Hiveserver2 会模拟成客户端的登录用户去访问 Hadoop 集群的数据，不启用，则 Hivesever2 会直接使用启动用户访问 Hadoop 集群数据。模拟用户的功能，默认是{==开启==}的。

具体逻辑如下：

<figure markdown>
  ![image-20230212223120563](https://cos.gump.cloud/uPic/image-20230212223120563.png)
  <figcaption>未开启用户模拟功能</figcaption>
</figure>

<figure markdown>
  ![image-20230212223131242](https://cos.gump.cloud/uPic/image-20230212223131242.png)
  <figcaption>开启用户模拟功能</figcaption>
</figure>
生产环境，推荐开启用户模拟功能，因为开启后才能保证各用户之间的权限隔离。

2）Hiveserver2 部署

（1）Hadoop 端配置

Hivesever2 的模拟用户功能，依赖于 Hadoop 提供的 proxy user（代理用户功能），只有 Hadoop 中的代理用户才能模拟其他用户的身份访问 Hadoop 集群。因此，需要{==将 Hiveserver2 的启动用户设置为 Hadoop 的代理用户==}，配置方式如下：

修改配置文件 core-site.xml，然后记得分发三台机器

```shell
cd $HADOOP_HOME/etc/hadoop
vim core-site.xml
```

```xml title="core-site.xml"
<!--配置所有节点的atguigu用户都可作为代理用户-->
<property>
    <name>hadoop.proxyuser.atguigu.hosts</name>
    <value>*</value>
</property>

<!--配置atguigu用户能够代理的用户组为任意组-->
<property>
    <name>hadoop.proxyuser.atguigu.groups</name>
    <value>*</value>
</property>

<!--配置atguigu用户能够代理的用户为任意用户-->
<property>
    <name>hadoop.proxyuser.atguigu.users</name>
    <value>*</value>
</property>
```

（2）Hive 端配置

在 hive-site.xml 文件中添加如下配置信息

```xml title="hive-site.xml"
<!-- 指定hiveserver2连接的host -->
<property>
	<name>hive.server2.thrift.bind.host</name>
	<value>hadoop102</value>
</property>

<!-- 指定hiveserver2连接的端口号 -->
<property>
	<name>hive.server2.thrift.port</name>
	<value>10000</value>
</property>
```

3）测试

（1）启动 Hiveserver2

```shell
bin/hive --service hiveserver2
```

（2）使用命令行客户端 beeline 进行远程访问

启动 beeline 客户端

```shell
 bin/beeline -u jdbc:hive2://hadoop102:10000 -n atguigu
```

看到如下界面

```shell
Connecting to jdbc:hive2://hadoop102:10000
Connected to: Apache Hive (version 3.1.3)
Driver: Hive JDBC (version 3.1.3)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 3.1.3 by Apache Hive
0: jdbc:hive2://hadoop102:10000>
```

（3）使用 Datagrip 图形化客户端进行远程访问

4）配置 DataGrip 连接

（1）创建连接

![image-20230212223558305](https://cos.gump.cloud/uPic/image-20230212223558305.png)

（2）配置连接属性

所有属性配置，和 Hive 的 beeline 客户端配置一致即可。初次使用，配置过程会提示缺少 JDBC 驱动，按照提示下载即可。

![image-20230212223623044](https://cos.gump.cloud/uPic/image-20230212223623044.png)

（3)界面介绍

![image-20230212223630255](https://cos.gump.cloud/uPic/image-20230212223630255.png)

（4）测试 sql 执行

![image-20230212223707688](https://cos.gump.cloud/uPic/image-20230212223707688.png)

（5）修改数据库

![image-20230212223717280](https://cos.gump.cloud/uPic/image-20230212223717280.png)

### 2.5.2 metastore服务

Hive 的 metastore 服务的作用是为 Hive CLI 或者 Hiveserver2 提供元数据访问接口。

1）metastore 运行模式

metastore 有两种运行模式，分别为嵌入式模式和独立服务模式。下面分别对两种模式进行说明：

（1）嵌入式模式

![image-20230212223854544](https://cos.gump.cloud/uPic/image-20230212223854544.png)

（2）独立服务模式

![image-20230212223828234](https://cos.gump.cloud/uPic/image-20230212223828234.png)

生产环境中，不推荐使用嵌入式模式。因为其存在以下两个问题：

（1）嵌入式模式下，每个 Hive CLI 都需要直接连接元数据库，当 Hive CLI 较多时，数据库压力会比较大。

（2）每个客户端都需要用户元数据库的读写权限，元数据库的安全得不到很好的保证。

2）metastore 部署

1）嵌入式模式

嵌入式模式下，只需保证 Hiveserver2 和每个 Hive CLI 的配置文件 hive-site.xml 中包含连接元数据库所需要的以下参数即可：

```xml title="hive-site.xml"
    <!-- jdbc连接的URL -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
    </property>
    
    <!-- jdbc连接的Driver-->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    
	<!-- jdbc连接的username-->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <!-- jdbc连接的password -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>
```

（2）独立服务模式

独立服务模式需做以下配置：

首先，保证 metastore 服务的配置文件 hive-site.xml 中包含连接元数据库所需的以下参数：

```xml title="hive-site.xml"
    <!-- jdbc连接的URL -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
    </property>
    
    <!-- jdbc连接的Driver-->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    
	<!-- jdbc连接的username-->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <!-- jdbc连接的password -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>
```

其次，保证 Hiveserver2 和每个 Hive CLI 的配置文件 hive-site.xml 中包含访问 metastore 服务所需的以下参数：

```xml title="hive-site.xml"
<!-- 指定metastore服务的地址 -->
<property>
	<name>hive.metastore.uris</name>
	<value>thrift://hadoop102:9083</value>
</property>
```

注意：主机名需要改为 metastore 服务所在节点，端口号无需修改，metastore 服务的默认端口就是 9083。

3）测试

此时启动 Hive CLI，执行 show databases 语句，会出现一下错误提示信息：

```shell
FAILED: HiveException java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
```

这是因为我们在 Hive CLI 的配置文件中配置了 hive.metastore.uris 参数，此时 Hive CLI 会去请求我们执行的 metastore 服务地址，所以必须启动 metastore 服务才能正常使用。

metastore 服务的启动命令如下：

```shell
hive --service metastore
```

!!! info "注意"

    启动后该窗口不能再操作，需打开一个新的 Xshell 窗口来对 Hive 操作。

重新启动 Hive CLI，并执行 show databases语句，就能正常访问了

```shell
bin/hive
```

### 2.5.3 编写Hive服务启动脚本（了解）

1）前台启动的方式导致需要打开多个 Xshell 窗口，可以使用如下方式后台方式启动

- nohup：放在命令开头，表示不挂起，也就是关闭终端进程也继续保持运行状态
- /dev/null：是 Linux 文件系统中的一个文件，被称为黑洞，所有写入该文件的内容都会被自动丢弃
- 2>&1：表示将错误重定向到标准输出上
- &：放在命令结尾，表示后台运行

一般会组合使用：`nohup  [xxx命令操作]> file  2>&1 &`，表示将 xxx 命令运行的结果输出到 file 中，并保持命令启动的进程在后台运行。

```shell
nohup hive --service metastore 2>&1 &
nohup hive --service hiveserver2 2>&1 &
```

2）为了方便使用，可以直接编写脚本来管理服务的启动和关闭

```shell
vim $HIVE_HOME/bin/hiveservices.sh
```

```sh title="hive services.sh"
#!/bin/bash

HIVE_LOG_DIR=$HIVE_HOME/logs
if [ ! -d $HIVE_LOG_DIR ]
then
	mkdir -p $HIVE_LOG_DIR
fi

#检查进程是否运行正常，参数1为进程名，参数2为进程端口
function check_process()
{
    pid=$(ps -ef 2>/dev/null | grep -v grep | grep -i $1 | awk '{print $2}')
    ppid=$(netstat -nltp 2>/dev/null | grep $2 | awk '{print $7}' | cut -d '/' -f 1)
    echo $pid
    [[ "$pid" =~ "$ppid" ]] && [ "$ppid" ] && return 0 || return 1
}

function hive_start()
{
    metapid=$(check_process HiveMetastore 9083)
    cmd="nohup hive --service metastore >$HIVE_LOG_DIR/metastore.log 2>&1 &"
    [ -z "$metapid" ] && eval $cmd || echo "Metastroe服务已启动"
    server2pid=$(check_process HiveServer2 10000)
    cmd="nohup hive --service hiveserver2 >$HIVE_LOG_DIR/hiveServer2.log 2>&1 &"
    [ -z "$server2pid" ] && eval $cmd || echo "HiveServer2服务已启动"
}

function hive_stop()
{
metapid=$(check_process HiveMetastore 9083)
    [ "$metapid" ] && kill $metapid || echo "Metastore服务未启动"
    server2pid=$(check_process HiveServer2 10000)
    [ "$server2pid" ] && kill $server2pid || echo "HiveServer2服务未启动"
}

case $1 in
"start")
    hive_start
    ;;
"stop")
    hive_stop
    ;;
"restart")
    hive_stop
    sleep 2
    hive_start
    ;;
"status")
    check_process HiveMetastore 9083 >/dev/null && echo "Metastore服务运行正常" || echo "Metastore服务运行异常"
    check_process HiveServer2 10000 >/dev/null && echo "HiveServer2服务运行正常" || echo "HiveServer2服务运行异常"
    ;;
*)
    echo Invalid Args!
    echo 'Usage: '$(basename $0)' start|stop|restart|status'
    ;;
esac
```

3）添加执行权限

```shell
chmod +x $HIVE_HOME/bin/hiveservices.sh
```

4）启动 Hive 后台服务

```shell
hiveservices.sh start
```

## 2.6 Hive使用技巧

### 2.6.1 Hive常用交互命令

```shell
bin/hive -help

usage: hive
 -d,--define <key=value>          Variable subsitution to apply to hive
                                  commands. e.g. -d A=B or --define A=B
    --database <databasename>     Specify the database to use
 -e <quoted-query-string>         SQL from command line
 -f <filename>                      SQL from files
 -H,--help                        Print help information
    --hiveconf <property=value>   Use value for given property
    --hivevar <key=value>         Variable subsitution to apply to hive
                                  commands. e.g. --hivevar A=B
 -i <filename>                    Initialization SQL file
 -S,--silent                      Silent mode in interactive shell
 -v,--verbose                     Verbose mode (echo executed SQL to the console)
```

1）在 Hive 命令行里创建一个表 student，并插入 1 条数据

```sql
create table student(id int,name string);
insert into table student values(1,"zhangsan");
select * from student;
```

2）`-e` 不进入 Hive 的交互窗口执行 hql 语句

```shell
bin/hive -e "select id from student;"
```

3）`-f` 执行脚本中的 hql 语句

（1）在 /opt/module/hive/ 下创建 datas 目录并在 datas 目录下创建 hivef.sql 文件

```shell
mkdir datas
vim hivef.sql
```

（2）文件中写入正确的 hql 语句

```sql title="hive.sql"
select * from student;
```

（3）执行文件中的 hql 语句

```shell
bin/hive -f /opt/module/hive/datas/hivef.sql
```

（4）执行文件中的 hql 语句并将结果写入文件中

```shell
bin/hive -f /opt/module/hive/datas/hivef.sql  > /opt/module/hive/datas/hive_result.txt
```

### 2.6.2 Hive参数配置方式

1）查看当前所有的配置信息

```sql
set;
```

2）参数的配置三种方式

（1）配置文件方式

- 默认配置文件：hive-default.xml
- 用户自定义配置文件：hive-site.xml

!!! info "注意"

    用户自定义配置会覆盖默认配置。另外，Hive 也会读入 Hadoop 的配置，因为 Hive 是作为 Hadoop 的客户端启动的，Hive 的配置会覆盖 Hadoop 的配置。配置文件的设定对本机启动的所有 Hive 进程都有效。

（2）命令行参数方式

①启动 Hive 时，可以在命令行添加 `-hiveconf param=value` 来设定参数。例如：

```shell
bin/hive -hiveconf mapreduce.job.reduces=10;
```

!!! warning "注意"

    仅对本次 Hive 启动有效。

②查看参数设置

```sql
set mapreduce.job.reduces;
```

（3）参数声明方式

可以在 HQL 中使用 SET 关键字设定参数，例如：

```sql
set mapreduce.job.reduces=10;
```
!!! warning "注意"

    仅对本次 Hive 启动有效。

查看参数设置：

```sql
set mapreduce.job.reduces;
```

上述三种设定方式的优先级依次递增。即 配置文件 < 命令行参数 < 参数声明。注意某些系统级的参数，例如 log4j 相关的设定，必须用前两种方式设定，因为那些参数的读取在会话建立以前已经完成了。

### 2.6.3 Hive常见属性配置

1）Hive 客户端显示当前库和表头

（1）在 hive-site.xml 中加入如下两个配置:

```xml title="hive-site.xml"
<property>
    <name>hive.cli.print.header</name>
    <value>true</value>
    <description>Whether to print the names of the columns in query output.</description>
</property>
<property>
    <name>hive.cli.print.current.db</name>
    <value>true</value>
    <description>Whether to include the current database in the Hive prompt.</description>
</property>
```

（2）Hive 客户端在运行时可以显示当前使用的库和表头信息

```shell
[atguigu@hadoop102 conf]$ hive

hive (default)> select * from stu;
OK
stu.id	stu.name
1	ss
Time taken: 1.874 seconds, Fetched: 1 row(s)
hive (default)>
```

2）Hive 运行日志路径配置

（1）Hive 的 log 默认存放在 /tmp/atguigu/hive.log 目录下（当前用户名下）

（2）修改 Hive 的 log 存放日志到 /opt/module/hive/logs

修改 $HIVE_HOME/conf/hive-log4j2.properties.template 文件名称为 hive-log4j2.properties

```shell
mv hive-log4j2.properties.template hive-log4j2.properties
```

在 hive-log4j2.properties 文件中修改 log 存放位置

```properties title="hive-log4j2.properties"
property.hive.log.dir=/opt/module/hive/logs
```

3）Hive 的 JVM 堆内存设置

新版本的 Hive 启动的时候，默认申请的 JVM 堆内存大小为 {==256M==}，JVM 堆内存申请的太小，导致后期开启本地模式，执行复杂的 SQL 时经常会报错：java.lang.OutOfMemoryError: Java heap space，因此最好提前调整一下 HADOOP_HEAPSIZE 这个参数。

（1）修改 $HIVE_HOME/conf 下的 hive-env.sh.template 为 hive-env.sh

```shell
mv hive-env.sh.template hive-env.sh
```

（2）将 hive-env.sh 其中的参数 export HADOOP_HEAPSIZE 修改为 2048，重启 Hive。

```sh title="hive-env.sh"
# The heap size of the jvm stared by hive shell script can be controlled via:
export HADOOP_HEAPSIZE=2048
```

4）关闭 Hadoop 虚拟内存检查

在 yarn-site.xml 中关闭虚拟内存检查（虚拟内存校验，如果已经关闭了，就不需要配了）。

（1）修改前记得先停 Hadoop

```shell
vim yarn-site.xml
```

（2）添加如下配置

```xml title="yarn-site.xml"
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
</property>
```

（3）修改完后记得分发 yarn-site.xml，并重启 yarn。
