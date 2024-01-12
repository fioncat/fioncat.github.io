---
layout:       post
title:        "Hadoop笔记一：伪分布式安装"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Hadoop
---

Hadoop安装分为单机、伪分布式和完全分布式。

- 单机模式是Hadoop的默认模式。在初次安装Hadoop后，将使用这个模式。此时Hadoop的三个配置文件为空。单机模式不使用HDFS，也不加载任何Hadoop守护进程，仅用来调试MapReduce程序。
- 伪分布式，Hadoop的守护进程在一台机器上运行，模拟一个小规模的集群。HDFS和MapReduce可以正常使用。可用于开发和生产前的调试。
- 完全分布式，Hadoop生产模式，完全运行在一个服务器集群上，需要搭建多台机器或虚拟机。用于真实部署。

## 安装前准备

下面演示如何在一台Linux主机(CentOS 6)上安装Hadoop。

注意要想使用Hadoop，系统必须配置了Java运行环境（最好直接上jdk）下面的命令展示了如何安装jdk。

```console
[root@hadoop1 ~]# mkdir /usr/develop
[root@hadoop1 ~]# tar -xzf jdk-8u65-linux-x64.tar.gz -C /usr/develop
[root@hadoop1 ~]# ls /usr/develop/
jdk1.8.0_65
```

随后将jdk的bin加到path中并且增加JAVA_HOME环境变量：

```console
[root@hadoop1 ~]# cd /usr/develop/jdk1.8.0_65/
[root@hadoop1 jdk1.8.0_65]# echo "export JAVA_HOME=\"$(pwd)\"" >> /etc/profile
[root@hadoop1 jdk1.8.0_65]# echo "export JRE_HOME=\"$(pwd)/jre\"" >> /etc/profile
[root@hadoop1 jdk1.8.0_65]# echo "export PATH=\"\$PATH:$(pwd)/bin\"" >> /etc/profile
[root@hadoop1 jdk1.8.0_65]# source /etc/profile
```

接着我们需要配置网络环境。

```console
[root@hadoop1 ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth0
```

将脚本文件改成如下样子：

```bash
DEVICE=eth0
BOOTPROTO=static
IPADDR=IP地址
NETMASK=255.255.255.0
GATEWAY=192.168.117.2
ONBOOT=yes
TYPE=Ethernet
IPV6INIT=no
DNS1=192.168.117.2
```

随后修改/etc/network和/etc/hosts文件，加上你的host配置：

```console
[root@hadoop1 ~]# vim /etc/sysconfig/network
  NETWORKING=yes
  HOSTNAME=hadoop1
[root@hadoop1 ~]# vim /etc/hosts
  127.0.0.1   localhost
  ::1         localhost
  192.168.117.51 hadoop1
```

在使用伪分布式时，我们可以生成公钥从而方便进行ssh免密登录（注意是自己免密登录自己）。

```console
[root@hadoop1 ~]# ssh-keygen
[root@hadoop1 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@hadoop1
```

随后可以验证以下可不可以实现免密登录：

```console
[root@hadoop1 ~]# ssh root@lazycat.hadoop1
Last login: Wed Mar 21 00:07:47 2018 from 192.168.117.1
[root@hadoop1 ~]# logout
Connection to lazycat.hadoop1 closed.
```

## 安装Hadoop

解压Hadoop的包：

```console
[root@hadoop1 ~]# tar zxf hadoop-2.7.1_64bit.tar.gz -C /usr/develop/
```

我们来看看Hadoop的目录结构：

```console
[root@hadoop1 hadoop-2.7.1]# ls
bin  etc  include  lib  libexec  LICENSE.txt  NOTICE.txt  README.txt  sbin  share
```

对目录进行一个简单的说明：

- bin: 存放脚本
- etc/hadoop: 存放hadoop配置文件
- lib: 存放运行时的依赖jar包
- sbin: 启动和关闭hadoop的命令
- libexec: 存放hadoop命令，一般不常用

## 配置伪分布式环境

下面配置Hadoop的环境变量，先修改hadoop-env.sh文件：

```console
[root@hadoop1 hadoop-2.7.1]# vim etc/hadoop/hadoop-env.sh
```

默认的JAVA_HOME和HADOOP_CONF_DIR是不好用的，需要我们自己配置成本机的绝对路径：

```bash
export JAVA_HOME="/usr/develop/jdk1.8.0_65"
export HADOOP_CONF_DIR="/usr/develop/hadoop-2.7.1/etc/hadoop"
```

如果不这样配置，后面在启动hadoop的时候可能会报错。

接下来，修改core-site.xml文件：

```console
[root@hadoop1 hadoop-2.7.1]# vim etc/hadoop/core-site.xml
```

需要指定NameNode(后面会详细介绍)，以及Hadoop运行时产生的临时文件(不配置默认存放到/tmp，可能会有安全隐患)。

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop1:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/develop/hadoop-2.7.1/tmp</value>
    </property>
</configuration>
```

接着，需要修改hdfs-site.xml：

```console
[root@hadoop1 hadoop-2.7.1]# vim etc/hadoop/hdfs-site.xml
```

我们需要指定hdfs保存副本的数量(默认为3)，在为分布情况下，这个值必须为1，另外，设置HDFS的操作权限，让任何用户可以操作HDFS：

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
     </property>
     <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
```

下面修改mapred-site.xml。这个配置文件在初始的时候是没有的，但是可以直接赋值模版文件：

```console
[root@hadoop1 hadoop-2.7.1]# cd etc/hadoop/
[root@hadoop1 hadoop]# cp mapred-site.xml.template mapred-site.xml
[root@hadoop1 hadoop]# vim mapred-site.xml
```

对于MapReduce的配置，我们需要让MapReduce运行在yarn上，这样可以提高MapReduce作业的效率：

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

既然MapReduce运行在yarn上，就需要配置yarn-site.xml，需要指定yarn的resourceManager和NodeManager的获取方式：

```console
[root@hadoop1 hadoop]# vim yarn-site.xml
```

具体配置如下：

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop1</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

接着，配置DataNode：

```console
[root@hadoop1 hadoop]# vim slaves
```

因为目前我们只有一台主机，所以在里面加上我们的主机名即可。

最后，需要配置Hadoop的环境变量，在/etc/profile加上：

```bash
export CLASSPATH=".:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tool.jar"
export HADOOP_HOME="/usr/develop/hadoop-2.7.1"
export PATH="$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin"
```

随后，就可以查看Hadoop的版本了：

```console
[root@hadoop1 hadoop]# hadoop version
Hadoop 2.7.1
Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r 15ecc87ccf4a0228f35af08fc56de536e6ce657a
Compiled by jenkins on 2015-06-29T06:04Z
Compiled with protoc 2.5.0
From source with checksum fc0a1a23fc1868e4d5ee7fa2b28a58a
This command was run using /usr/develop/hadoop-2.7.1/share/hadoop/common/hadoop-common-2.7.1.jar
```

执行format命令，可以格式化NameNode：

```console
[root@hadoop1 hadoop-2.7.1]# hadoop namenode -format
```

随后，启动Hadoop的监听程序：

```console
[root@hadoop1 hadoop-2.7.1]# cd sbin/
[root@hadoop1 sbin]# ./start-all.sh
```

start-all会启动所有服务(hdfs和yarn)，sbin里面有很多脚本，可以只启动特定的服务。

使用JPS命令，可以看到Hadoop守护进程的id：

```console
[root@hadoop1 sbin]# jps
2128 NameNode
2562 ResourceManager
2659 NodeManager
2251 DataNode
2412 SecondaryNameNode
3022 Jps
```

这些服务的具体功能后面会详细介绍。使用stop-all.sh会关闭所有Hadoop守护进程。

可以利用netstat观察到监听程序的信息，例如，看NameNode的信息：

```console
[root@hadoop1 hadoop-2.7.1]# netstat -tpln | grep 2128
tcp        0      0 0.0.0.0:50070               0.0.0.0:*                   LISTEN      2128/java
tcp        0      0 192.168.117.51:9000         0.0.0.0:*                   LISTEN      2128/java
```

可见它监听了一个50070端口，我们在浏览器可以利用这个端口访问Hadoop的图形工具：http://ip:50070/

利用这个工具可以简单查看NameNode信息。

注意如果Hadoop报警告：

```log4j
18/03/21 04:28:42 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 1 items
```

这个警告事实上没有什么影响，我们可以粗暴地关闭它。

```console
[root@hadoop1 hadoop-2.7.1]# vim etc/hadoop/log4j.properties
```

在文件中添加：

```properties
log4j.logger.org.apache.hadoop.util.NativeCodeLoader=ERROR
```

那么，警告就消失了。（具体解决方法请自行查询资料，这里不再叙述）
