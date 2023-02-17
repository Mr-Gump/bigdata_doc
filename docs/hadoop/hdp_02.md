# 第2章 Hadoop运行环境搭建（开发重点）

## 2.1 模板虚拟机环境准备

0）安装模板虚拟机，IP 地址 192.168.10.100、主机名称 hadoop100、内存 4G、硬盘 50G

1）hadoop100 虚拟机配置要求如下（本文 Linux 系统全部以 CentOS-7.5-x86-1804 为例）

（1）使用 yum 安装需要虚拟机可以正常上网，yum 安装前可以先测试下虚拟机联网情况

（2）安装 epel-release

!!! info "注"

    Extra Packages for Enterprise Linux 是为“红帽系”的操作系统提供额外的软件包，适用于 RHEL、CentOS 和 Scientific Linux。相当于是一个软件仓库，大多数 rpm 包在官方 repository 中是找不到的）

```shell
yum install -y epel-release
```

（3）注意：如果 Linux 安装的是最小系统版，还需要安装如下工具；如果安装的是 Linux 桌面标准版，不需要执行如下操作

net-tool：工具包集合，包含 ifconfig 等命令

```shell
yum install -y net-tools 
```

vim：编辑器

```shell
yum install -y vim
```

一些其他工具

```shell
yum install -y  net-tools vim psmisc  nc  rsync  lrzsz  ntp libzstd openssl-static tree iotop git 
```

2）关闭防火墙，关闭防火墙开机自启

```shell
systemctl stop firewalld
systemctl disable firewalld.service
```

!!! info "注意"

    在企业开发时，通常单个服务器的防火墙时关闭的。公司整体对外会设置非常安全的防火墙

3）创建 atguigu 用户，并修改 atguigu 用户的密码

```shell
useradd atguigu
passwd atguigu
```

4）配置 atguigu 用户具有 root 权限，方便后期加 sudo 执行 root 权限的命令

```txt title="sudoers"
## Allow root to run any commands anywhere
root    ALL=(ALL)     ALL
atguigu   ALL=(ALL)     NOPASSWD:ALL
```

5）在 /opt 目录下创建文件夹，并修改所属主和所属组

（1）在 /opt 目录下创建 module、software 文件夹

```shell
mkdir /opt/module
mkdir /opt/software
```

​	（2）修改 module、software 文件夹的所有者和所属组均为 atguigu 用户 

```shell
chown atguigu:atguigu /opt/module 
chown atguigu:atguigu /opt/software
```

（3）查看 module、software 文件夹的所有者和所属组

```shell
ll

总用量 12
drwxr-xr-x. 2 atguigu atguigu 4096 5月  28 17:18 module
drwxr-xr-x. 2 root    root    4096 9月   7 2017 rh
drwxr-xr-x. 2 atguigu atguigu 4096 5月  28 17:18 software
```

6）卸载虚拟机自带的 JDK

!!!	info "注意"

    如果你的虚拟机是最小化安装不需要执行这一步。

```shell
rpm -qa | grep -i java | xargs -n1 rpm -e --nodeps 
```

- rpm -qa：查询所安装的所有rpm软件包
- grep -i：忽略大小写
- xargs -n1：表示每次只传递一个参数
- rpm -e –nodeps：强制卸载软件

7）重启虚拟机

```shell
reboot
```

## 2.2 克隆虚拟机

1）利用模板机 hadoop100，克隆三台虚拟机：hadoop102 hadoop103 hadoop104

!!! info "注意"

    克隆时，要先关闭 hadoop100

2）修改克隆机 IP，以下以 hadoop102 举例说明

（1）修改克隆虚拟机的静态 IP

```shell
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

改成

```txt title="ifcfg-ens33"
DEVICE=ens33
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static
NAME="ens33"
IPADDR=192.168.10.102
PREFIX=24
GATEWAY=192.168.10.2
DNS1=192.168.10.2
```

（2）查看 Linux 虚拟机的虚拟网络编辑器，编辑 -> 虚拟网络编辑器- > VMnet8

![image-20230217224442439](https://cos.gump.cloud/uPic/image-20230217224442439.png)

![image-20230217224448197](https://cos.gump.cloud/uPic/image-20230217224448197.png)

（3）查看 Windows 系统适配器 VMware Network Adapter VMnet8 的 IP 地址

![image-20230217224509908](https://cos.gump.cloud/uPic/image-20230217224509908.png)

（4）保证 Linux 系统 ifcfg-ens33 文件中 IP 地址、虚拟网络编辑器地址和 Windows 系统 VM8 网络 IP 地址相同。

3）修改克隆机主机名，以下以 hadoop102 举例说明

​	（1）修改主机名称

```shell
vim /etc/hostname
```

```txt title="hostname"
hadoop102
```

（2）配置 Linux 克隆机主机名称映射 hosts 文件，打开 /etc/hosts

```shell
vim /etc/hosts
```

```txt title="hosts"
192.168.10.100 hadoop100
192.168.10.101 hadoop101
192.168.10.102 hadoop102
192.168.10.103 hadoop103
192.168.10.104 hadoop104
192.168.10.105 hadoop105
192.168.10.106 hadoop106
192.168.10.107 hadoop107
```

4）重启克隆机 hadoop102 

```shell
reboot
```

5）修改 windows 的主机映射文件（ hosts 文件）

（1）如果操作系统是 window7，可以直接修改 

①进入 C:\Windows\System32\drivers\etc 路径

②打开 hosts 文件并添加如下内容，然后保存

```txt title="hosts"
192.168.10.100 hadoop100
192.168.10.101 hadoop101
192.168.10.102 hadoop102
192.168.10.103 hadoop103
192.168.10.104 hadoop104
192.168.10.105 hadoop105
192.168.10.106 hadoop106
192.168.10.107 hadoop107
192.168.10.108 hadoop108
```

（2）如果操作系统是 window10，先拷贝出来，修改保存以后，再覆盖即可

①进入 C:\Windows\System32\drivers\etc 路径

②拷贝 hosts 文件到桌面

③打开桌面 hosts 文件并添加如下内容

④将桌面 hosts 文件覆盖 C:\Windows\System32\drivers\etc 路径 hosts 文件

## 2.3 在hadoop102安装JDK

1）卸载现有 JDK

2）用 XShell 传输工具将 JDK 导入到 opt 目录下面的 software 文件夹下面

![image-20230217224914309](https://cos.gump.cloud/uPic/image-20230217224914309.png)

3）在 Linux 系统下的 opt 目录中查看软件包是否导入成功

```shell
ls /opt/software/
```

4）解压 JDK 到 /opt/module 目录下

```shell
tar -zxvf jdk-8u212-linux-x64.tar.gz -C /opt/module/
```

5）配置 JDK 环境变量

（1）新建 /etc/profile.d/my_env.sh 文件

```shell
sudo vim /etc/profile.d/my_env.sh
```

```sh title="my_env.sh"
#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_212
export PATH=$PATH:$JAVA_HOME/bin
```

​	（2）保存后退出

​	（3）source 一下 /etc/profile 文件，让新的环境变量 PATH 生效

```shell
source /etc/profile
```

6）测试 JDK 是否安装成功

```shell
java -version
```

如果能看到以下结果，则代表 Java 安装成功。

```shell
java version "1.8.0_212"
```

## 2.4 在hadoop102安装Hadoop

[:link:Hadoop下载地址](https://archive.apache.org/dist/hadoop/common/hadoop-3.1.3/)

1）用 XShell 文件传输工具将 hadoop-3.1.3.tar.gz 导入到 opt 目录下面的 software 文件夹下面

![image-20230217225216574](https://cos.gump.cloud/uPic/image-20230217225216574.png)

2）进入到 Hadoop 安装包路径下

```shell
cd /opt/software/
```

3）解压安装文件到 /opt/module 下面

```shell
tar -zxvf hadoop-3.1.3.tar.gz -C /opt/module/
```

4）查看是否解压成功

```shell
ls /opt/module/

hadoop-3.1.3
```

5）将 Hadoop 添加到环境变量

​	（1）获取 Hadoop 安装路径

```shell
pwd

/opt/module/hadoop-3.1.3
```

​	（2）打开 /etc/profile.d/my_env.sh 文件

```sh title="my_env.sh"
#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

​	（3）让修改后的文件生效

```shell
source /etc/profile
```

6）测试是否安装成功

```shell
hadoop version

Hadoop 3.1.3
```

7）重启（如果 Hadoop 命令不能用再重启虚拟机）

```shell
sudo reboot
```

## 2.5 Hadoop目录结构

1）查看 Hadoop 目录结构

```shell
ll

总用量 52
drwxr-xr-x. 2 atguigu atguigu  4096 5月  22 2017 bin
drwxr-xr-x. 3 atguigu atguigu  4096 5月  22 2017 etc
drwxr-xr-x. 2 atguigu atguigu  4096 5月  22 2017 include
drwxr-xr-x. 3 atguigu atguigu  4096 5月  22 2017 lib
drwxr-xr-x. 2 atguigu atguigu  4096 5月  22 2017 libexec
-rw-r--r--. 1 atguigu atguigu 15429 5月  22 2017 LICENSE.txt
-rw-r--r--. 1 atguigu atguigu   101 5月  22 2017 NOTICE.txt
-rw-r--r--. 1 atguigu atguigu  1366 5月  22 2017 README.txt
drwxr-xr-x. 2 atguigu atguigu  4096 5月  22 2017 sbin
drwxr-xr-x. 4 atguigu atguigu  4096 5月  22 2017 share
```

2）重要目录

（1）bin 目录：存放对 Hadoop 相关服务（hdfs，yarn，mapred）进行操作的脚本

（2）etc 目录：Hadoop 的配置文件目录，存放 Hadoop 的配置文件

（3）lib 目录：存放 Hadoop 的本地库（对数据进行压缩解压缩功能）

（4）sbin 目录：存放启动或停止 Hadoop 相关服务的脚本

（5）share 目录：存放 Hadoop 的依赖 jar 包、文档、和官方案例

