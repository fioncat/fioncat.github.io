---
layout:       post
title:        "HBase 基础笔记"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - HBase
---

HBase是基于Hadoop的一款数据库工具。它来源于Google的一篇论文BigTable。后来由Apache做了开源实现，就是HBase。

HBase是一种NoSQL(非关系型数据库)。适合储存非结构化和半结构化的数据，适合储存稀疏的数据(空的数据不占据空间)，HBase是面向列(族)储存的。在底层是按照列为单位进行数据储存的。

不同于Hive，即使HBase是基于HDFS的，它仍然可以做到增删改查。并且可以储存海量数据，性能非常强大。可以实现上亿条数据的毫秒级别的查询(常见的RDBMS如MySQL的性能瓶颈仅千万)。

但是，在事务方面，HBase表现不如RDBMS。不支持表级别的事务。

HBase利用HDFS作为文件的储存系统，利用MapReduce处理海量数据，利用Zookeeper作为协调工具。这意味HBase是高可靠，高性能，可伸缩的分布式储存系统。

HBase有以下的基础概念：

- 行键，RowKey。即HBase的主键，可以有多个。查询的时候只能通过HBase的行键或者全表扫描进行查询。行键默认是按照字典顺序排序好的。
- 列族，Column Family。表的元数据，用于保存多个列。需要在建表的时候声明，不能在后期添加。
- 列，Column。列族中的列，不同于列族，列不需要提前声明，而且可以动态增加。
- 单元格和时间戳，Cell and Timestamp。通过row和column可以确定一个储存单元。每个储存单元都保存一个数据的多个版本，版本通过时间戳来区分。也就是说如果你改变了一个Cell中的数据，以前的数据不会被删除，而是根据时间戳进行版本区分。并且在单元格中数据都是以二进制的方式储存，没有所谓的数据类型区分。但是在取出数据的时候需要手动转换。

## 安装HBase

HBase安装必须需要jdk和Hadoop。HBase有常见的三个版本，下面这张表展示了3个版本适用的Hadoop版本：

Hadoop version | HBase-0.92.x | HBase-0.94.x | HBase-0.96
---|---|---| -- |
0.20.205 | 支持 | 不支持 | 不支持
0.22.x | 支持 | 不支持 | 不支持
0.23.x | 不支持 | 支持 | 未测试
1.0.x | 支持 | 支持 | 支持
1.1.x | 未测试 | 支持 | 支持
2.x | 不支持 | 支持 | 支持

下面我用Hadoop 2.7和HBase 0.96作为演示。

HBase的安装分为单机，伪分布式和完全分布式。单机模式把数据储存在本地，伪分布式把数据储存于HDFS，仅有一台HBase机器，完全分布式有多台HBase机器作为集群。

### 单机模式

首先，解压HBase：

```text
[root@hadoop1 mysql]# tar xzf hbase-0.98.17-hadoop2-bin.tar.gz -C /usr/develop/
[root@hadoop1 mysql]# cd /usr/develop/
[root@hadoop1 develop]# mv hbase-0.98.17-hadoop2/ hbase0.98
```

随后，需要更改HBase的核心配置文件conf/hbase-site.xml。在单机模式下，一般使用默认配置，但是需要增加一个简单的配置：

```xml
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>file:///usr/develop/hbase0.98/tmp</value>
    </property>
</configuration>
```

这个配置是指定HBase的数据储存路径，否则会储存到/tmp下，这样会导致HBase数据间歇性被系统删除。

随后启动HBase的守护进程：

```text
[root@hadoop1 hbase0.98]# bin/start-hbase.sh
starting master, logging to /usr/develop/hbase0.98/bin/../logs/hbase-root-master-hadoop1.out
[root@hadoop1 hbase0.98]# jps
6000 Jps
3552 NodeManager
5908 HMaster
3445 ResourceManager
3256 SecondaryNameNode
2953 NameNode
3053 DataNode
```

看到HMaster进程表示启动成功。在浏览器输入ip:60010，可以通过图形化界面管理HBase。

这样的HBase底层将数据储存在本地的FS。未使用到HDFS储存数据，一般只用于测试和开发。

### 伪分布式安装

伪分布式HBase更加类似于生产，它把数据储存在HDFS上。

要想实现伪分布式模式，需要修改hbase-site.xml配置文件：

```xml
<configuration>

    <!-- 将数据储存在HDFS -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://hadoop1:9000/hbase</value>
    </property>

    <!-- 声明每个数据块只有一份备份数据, 完全分布式此处应该和HDFS集群一致 -->
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>

</configuration>
```

如果电脑里头HDFS是开着的，那么就像单机模式一样启动即可。如果HDFS没有开则先需要启动HDFS。

### 完全分布式安装

完全分布式需要准备3台机器，作为HBase集群。这里克隆3台虚拟机即可。克隆的具体流程我不再演示。

完全分布式需要在hbase-site.xml中声明开启分布式，配置如下：

```xml
<property>
    <name>hbase.rootdir</name>
    <value>hdfs://hadoop1:9000/hbase</value>
</property>
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>
<property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
</property>
<property>
    <name>hbase.zookeeper.quorum</name>
    <value>hadoop1:2181,hadoop2:2181,hadoop3:2181</value>
</property>
```

注意HBase集群需要Zookeeper作为集群的协调工具。所以需要在3台机器安装Zookeeper并且启动2181监听程序。

我们需要修改conf/hbase-env.sh禁用ZooKeeper自动启动和销毁。如果这个选项设置为true(默认)，每当HBase启动，Zookeeper也会自动启动，HBase结束也会把ZooKeeper关闭。如果Zookeeper参与了其它集群的管理，显然这样是不合适的。

同时需要在这个文件中声明JAVA_HOME：

```bash
export HBASE_MANAGES_ZK=false
export JAVA_HOME=xxx
```

随后需要配置region服务器，我们修改conf/regionservers文件，在里面配置所有HBase主机，每台主机占据一行，这样当启动HBase的时候会自动启动所有节点的HBase监听程序(按照你配置的顺序)：

```text
[root@hadoop1 hbase0.98]# echo hadoop1 > conf/regionservers
[root@hadoop1 hbase0.98]# echo hadoop2 >> conf/regionservers
[root@hadoop1 hbase0.98]# echo hadoop3 >> conf/regionservers
[root@hadoop1 hbase0.98]# cat conf/regionservers
hadoop1
hadoop2
hadoop3
```

接着需要把HBase拷贝到hadoop1和hadoop2节点上。

随后就可以在一个节点上启动HBase了：

```text
[root@hadoop1 hbase0.98]# bin/start-hbase.sh
hadoop3: starting regionserver, logging to /usr/develop/hbase0.98/bin/../logs/hbase-root-regionserver-hadoop2.out
hadoop1: starting regionserver, logging to /usr/develop/hbase0.98/bin/../logs/hbase-root-regionserver-hadoop1.out
hadoop3: starting regionserver, logging to /usr/develop/hbase0.98/bin/../logs/hbase-root-regionserver-hadoop3.out
[root@hadoop1 hbase0.98]# jps
6161 Jps
5937 HRegionServer
2338 DataNode
4531 NodeManager
5028 QuorumPeerMain
2231 NameNode
4424 ResourceManager
2504 SecondaryNameNode
5595 HMaster
```

我们可以换一个机器，启用备用master，这样在主master挂掉的情况下这个master能够顶替：

```text
[root@hadoop2 hbase0.98]# bin/hbase-daemon.sh start master
starting master, logging to /usr/develop/hbase0.98/bin/../logs/hbase-root-master-hadoop2.out
[root@hadoop2 hbase0.98]# jps
4056 HMaster
3867 HRegionServer
3246 QuorumPeerMain
4094 Jps
```

这是HBase自带的高可靠，不需要我们做什么额外配置。

在安装好以后，运行bin/下的hbase，传入shell参数即可进入HBase的Shell界面：

```text
[root@hadoop2 hbase0.98]# cd bin/
[root@hadoop2 bin]# ./hbase shell
2018-04-11 21:23:03,302 INFO  [main] Configuration.deprecation: hadoop.native.lib is deprecated. Instead, use io.native.lib.available
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.98.17-hadoop2, rd5f8300c082a75ce8edbbe08b66f077e7d663a4a, Fri Jan 15 22:46:43 PST 2016

hbase(main):001:0>
```

## HBase命令

以下是HBase的常见命令：

命令 | 说明
----|----
通用命令|
status | 列出当前HBase的状态
version | 查看HBase的版本
whoami | 列出当前用户
DDL|
list | 列出当前的所有表
create | 创建一张表
describe | 显示一张表的信息
namespace|
alter_namespace | 修改namespace
list_namespaces | 列出namespace
create_namespace | 创建namespace
list_namespace_tables | 列出一个namespace下的所有表
drop_namespace | 删除namespace
DML|
disable | 禁用一张表
enable | 启用一张表
append | 追加数据
count | 统计数据行数
delete | 删除数据
deleteall | 删除所有数据
truncate | 摧毁表，重新建立一张一样的表，数据量大时效率比deleteall高很多
put | 向指定位置(单元格)储存值
scan | 查看一张表的所有数据
get | 取出某一具体行，列或者单元格的数据
drop | 删除一张表，在操作前需要disable这张表

### create 命令

利用create可以创建一张表。需要传入列族(至少一个)，列族的声明可以是简单的名称或者是一个map。

表名称可以指定名称空间。使用"namespace:tableName"的语法。

以下是create的一些常见用法：

```text
create 'namespace:table', {NAME => 'f1', VERSIONS => 5}
```

这创建了一张"table"表，位于"namespace"名称空间中。name如果不写默认为default。其中有一个列族，这个列族的名称为"f1"。版本是5。这个版本表示当数据更新时，版本会从1开始递增，当版本达到5以后，旧数据会被真正删除。

我们可以在{}中指定更多的属性。

可以声明多个列族：

```text
create 'namespace:table', {NAME => 'f1'}, {NAME => 'f2'}, {NAME => 'f3'}
```

VERSIONS不指定默认为1。以上写法可以简化为：

```text
create 'namespace:table', 'f1', 'f2', 'f3'
```

### put 命令

put可以向HBase的一张表的一个单元格插入数据（可以修改数据）。需要指定这个数据的行，列或者是时间戳(可选)。例如：

```text
put 'ns:t', 'row', 'column', 'value' [,timestamp]
```

row表示行键，它在整个表中是唯一的。

不指定时间戳将会使用当前时间。

注意column需要指明列族，用法是"cf:c"。

row和column不存在的话都会自动创建。

### get命令

get可以取出一个单元格的内容。必须指明行键，列可以不指明，表示查看这一行的所有内容。

```text
get 'ns:t' , 'row' [, 'column']
```

不传column则查看所有列。

这里的column使用的是列名称，你也可以传一个完整的map：

```text
get 'ns:t' , 'row' [, {COLUMN=>'r'}]
```

你也可以查看多个列：

```text
get 'ns:t', 'row', [, {COLUMN=>['c1', 'c2', ...]}]
```

以上写法可以简化为：

```text
get 'ns:t', 'row', ['c1', 'c2']
```

或者：

```text
get 'ns:t', 'row', 'c1', 'c2'
```

在查看列的时候，你也可以指明某个特定的时间戳或VERSIONS：

```text
get 'ns:t', 'row', [, {COLUMN=>'c', [,TIMESTAMP=t][,VERSIONS=v]}]
```

delete的用法和put类似，利用delete可以删除一个单元格的数据。

## HBase 原理

### 物理储存

HBase会把表划分为多个Region。Region按照行的方向排序。一个Region可能储存了多个行键。在一开始，一张表只有一个Region，在往表里头插入数据的时候，Region会越来越大。当大到预值后，会将原来的Region一分为二，变成两个Region。

Region是HBase中分布式储存和负载的最小单元。HBase集群的不同机器会储存不同的Region。储存Region的单元被叫做RegionServer。一个RegionServer可能储存了很多Region。

一个Region内部，有一个或者多个Store，一个列族对应一个Store，每个Store内部有一个memStore，有可能有0个，一个或多个storeFile。

memStore是储存在内存中的数据，storeFile是储存在HDFS中的文件。

### 读写数据

因为行键是排好序的，所有当有一个新的数据的时候，HBase会先计算出这个行键是由哪个Region储存的。然后根据这个Region找到对应的RegionServer的位置，然后根据列族找到对应的Store。

Region分为两个部分，一部分储存在内存中，一部分储存在HDFS中。HBase会先在内存中写入数据。但是内存的数据容易丢失并且内存容量是比较低的。在memStore快要满的时候，HBase会新开一个memStore，开一个新的线程将旧的memStore写入HDFS，在这个过程中发生的写操作会写到新的memStore中。

也就是说，每次对HBase的写入操作都是往内存中写的。中间向HDFS中做持久化的操作对于我们来说是透明的。所以HBase的写入效率是非常高的。

我们知道，HDFS不允许修改数据。所以数据一旦被存到HDFS里头，就很难被HBase改动了。所以假如此时客户端向HBase提出要修改数据，HBase并不会去动HDFS中的旧数据，而是把新数据写入memStore里头。这样在后面memStore被持久化为storeFile以后，HDFS中事实上储存了两份这个单元格的数据。后期查询的时候只能返回时间戳比较新的那个。

这会带来一个问题，如果HBase使用了很久，HDFS中就可能存在大量的垃圾数据无法被清理。

为了解决这个问题，HBase如果发觉StoreFile过多了，会尝试把多个storeFile合并为一个storeFile（单独启动一个线程做，不会影响数据写入）。在这个过程中就会把单元格中没有用的数据删除。从而减少垃圾数据。

合并过多次之后，storeFile会变得非常大，这对于查询不是很有利。所以当HBase发现storeFile太大时，就会把storeFile进行一次拆分。这时拆分后的文件已经没有垃圾数据了。

HBase这样设计非常精巧，也就是说对HBase的数据写入操作是完全基于内存的，用户不会被HDFS的性能瓶颈所限制。

如果HBase集群节点断电，memStore的数据会丢失，但是storeFile的数据不会丢失。

HBase为了解决这样的问题，会维护一个HLog文件(储存于HDFS)。HBase在往memStore写入数据之前，会在HLog中记录下这次操作行为，在修改完后报告成功。这样在断电以后，可以通过HLog找到操作日志，通过这个日志恢复memStore中的数据。

每当从memStore向HDFS中写数据的时候，会到ZK中记录下当前最后持久化的日志编号。这样当HBase挂掉后，HBase会找到最后一个持久化编号，并从这个编号后面开始恢复数据。

当HLog达到一定的大小后，HBase会删除掉一些已经持久化的日志记录。

每个RegionServer会储存一个HLog。

在读取的时候，如果数据在内存中，则直接读取。如果数据在HDFS中，则需要到HDFS中寻找这个行键的所有数据（可能会找到垃圾数据），把这些数据读到内存中做一个合并后输出。注意这个读取过程可能会扫描多个的storeFile，效率不是那么的高。但是也能基本保证和HDFS的读效率持平。

总之，HBase的写入效率相当于内存的写入效率，这点基本不会改变。而读取效率则不够稳定，有时候相当于内存的读取效率，有时候相当于HDFS的读取效率。然而由于HBase内部的索引优化，这样的效率差距还是能够让人接受的。

### StoreFile 结构

StoreFile是HBase储存在HDFS中的一个文件，它储存了表中的一些数据，这个文件由下面的6部分组成：

1. Data Block: 保存表中的数据，可以被压缩
2. Meta Block: 保存用户自定义key-value对，可以被压缩
3. File Info: StoreFile的元信息，不可以被压缩，可以定制
4. Data Block Index: Data Block的索引
5. Meta Block Index: Meta Block的索引
6. Traller: 定长的，保存了每个块的偏移量。在读取数据的时候，首先读取这一段数据，从而获知每块数据的位置。

DataBlock会被细分为多个子block，子block中储存了多个键值对，这是真正的数据。每个子DataBlock储存了一定范围的行键数据。在查询的时候，会先根据Traller获取Data Block Index和Data Block的位置。根据Index可以知道当前行键数据位于DataBlock的哪个具体子DataBlock中，然后把整个子DataBlock的数据返回到内存中合并。

因为对某个行键的查询可能从多个StoreFile中获取多个范围一样的DataBlock，所以获取之后要放到内存中做合并后再进行输出。

HBase的数据最终是按照键值对储存的。其中，key是行键+列族+列，value是具体的数据。key是按照行键的字典数据排序好的，这也是为什么HBase的空单元格不占据空间。

### Region 寻址

我们知道，HBase可以通过行键找到行键对应的Region，从而找到哪个RegionServer储存了这个行键的数据，那么HBase是如何实现这一点的?

Region是按照行键排好序的，所以确定一个行键储存在哪个Region中并不复杂。但是如何确定Region保存在哪台机器？

在HBase的hbase名称空间下有一张meta表，其中存放了Region和RegionServer的对应关系，也就是Region储存在哪台机器上。这个meta表只能够有一个Region。

当我们需要往一个Region储存数据的时候，会从这个meta表找到这个Region储存在具体的哪台机器上。meta的Region位置不是固定的。如果储存这个Region的机器挂掉，会将这个Region在其它机器恢复。

Region的位置由ZK记录。

下面梳理一遍往HBase中写入或者查询数据的完整流程：

- 客户端从ZK获取meta表Region的位置，并计算出要插入或修改数据的行键在哪个Region上。
- 找到meta表，查出这个Region储存在Hbase集群的哪台机器上。通过列族找到Region的Store。
- 如果是写入，直接在这台机器对应Region的memStore写入数据；如果是查询，首先到memStore查询，未命中则前往fileStore查询，并把多个fileStore查询的结果在内存合并再返回。

第一步并不是一直要做的。客户端可以缓存Region的位置信息，从而避免每次操作都需要访问ZK导致性能开销。

### 储存结构

一般认为，储存数据存在三种方式：

- Hash储存: 通过持久化哈希表实现数据储存。支持增删改以及随机读取(随机读取是它的优势)，但是不支持顺序扫描。也就是说哈希表的插入和查询操作效率特别高(比树高了一个数量级，O(1) vs O(n))。但是数据的排序和有序遍历效率特别低。
- B树储存: 通过持久化B树或其改进的树型数据结构(B+树等)实现对数据的储存。支持行级别的增删改，但是效率不及哈希表。支持顺序遍历，排序效率较高。传统的RDBMS(例如MySQL)一般使用这种数据结构。
- LSM树储存: Log-Struct Merge Tree, 日志储存合并树。HBase的储存方式。将一部分数据储存在内存中，一部分数据储存在磁盘里。优点是一次性批量将数据从内存写入磁盘从而大量减少了磁盘随机IO。LSM的写性能优于前两种储存方式(内存IO + 批量顺序磁盘IO vs 大量随机磁盘IO)，但是读效率不及前两种储存方式(读取效率不稳定，并且读取的时候可能需要涉及大量的merge操作)

也就是说，LSM通过牺牲读取效率换取了写效率的提升。

LSM树将数据优先储存在内存中，一旦内存不够了，就把数据储存到磁盘中去。在数据更新的时候，LSM为了高效的写操作，不会到磁盘中删除旧的数据，而是保存数据的多份副本。所以写入数据永远是面向内存的，效率特别高。

但是在读取数据的时候，如果没有命中内存，可能需要进行随机磁盘IO并且要将不同历史版本的数据进行合并，带来的额外的性能开销。在极端情况下，LSM的读取数据效率比MySQL低了一个数量级，但是写效率比MySQL又高了一个数量级。

在底层，LSM维护了若干个“小树”，这些树一开始储存在内存中，当树长到一定程度，会被持久化到磁盘中。LSM需要定期对磁盘中的小树进行merge，形成一个大树，又定期把这个大树进行拆分，又变成多个小树。

因为内存是不可靠的，不持久的，所以LSM要求有一个可靠的日志文件，记录每次操作，特别是持久化操作，方便内存数据丢失的恢复。

## HBase 常见问题

### HBase到底为什么这么快？

- 行键是排好序的，将整张表按照行键拆分成多个Region，储存在不同的机器上。这样在查询的时候，能够快速定位出数据在哪个机器上。并且这样的分布式储存能够进行多台机器的并发查询。
- 使用LSM树，将一部分数据保存在内存里头，在写入数据的时候能够基于内存写入。效率非常高。

### HBase为什么可以储存海量数据？

- 底层基于HDFS，这个分布式文件系统本来就是针对大数据设计的。
- 底层储存键值对，不储存表的结构，空的单元格不占据空间，对于稀疏表的空间利用率较高。
- 按照列进行储存，因为列的数据类型一般是一样的，所以底层可以对列数据进行压缩，节省空间。

### HBase可靠性如何？

- 本身基于ZK实现了HA。
- 底层储存于HDFS，有HDFS天生高可靠支持。
- HLog使得即使储存在内存的数据丢失了，也能够及时恢复。

### HBase vs Hive

- Hive仅仅是一个Hadoop的客户端。作为一个数据仓库使用。它的作用是利用SQL方便地对HDFS中的数据进行管理，能够编写MR任务。
- Hive一般用于离线分析，延时较高。并且无法进行行级别的增删改。
- HBase是真正意义上的数据库，提供了服务端和客户端。HDFS仅作为HBase的储存空间。并且HBase本身是分布式的。
- HBase可用作实时分析，写入、查询效率较高，并且支持行级别的增删改。

### HBase vs RDBMS

- RDBMS不适合储存海量数据，HBase因为是分布式的并且基于HDFS，适合储存海量数据。
- RDBMS适合储存结构化数据，固定表的结构，不允许增加列(事实上可以实现，但是可能需要重构表结构，效率很差，并且有一些限制)，底层按照行进行储存。HBase适合储存非结构化数据和半结构化数据，不固定表的结构，允许增加列，底层按照列进行储存。
- RDBMS支持非常复杂的结构化查询，HBase仅支持全表扫描和按照行键查询。
- RDBMS支持ACID事务，适合应用级别的数据储存；HBase不支持ACID事务，仅支持行级别的事务，不适合储存应用数据，适合储存日志、爬虫结果等辅助性的，用于数据分析的海量数据。

## Java API

我们不能使用JDBC来操作HBase，需要用到HBase自己的API。

我们需要使用以下的几个类来操作HBase：

- HBaseAdmin: 表示整个数据库对象。
- HBaseConfiguration: 数据库配置。
- HTable: 表
- HTableDescriptor: 表描述
- Put: 用于写入数据
- Get: 用于查询数据
- Scanner: 用于扫描表

下面是连接HBase并且创建表的代码：

```java
package cn.lazycat.bdd.hbase;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.HBaseAdmin;

import java.io.IOException;

public class HBaseTest {

    public static void main(String[] args) throws IOException {
        // 取得Conf
        Configuration conf = HBaseConfiguration.create();
        // 配置zk地址
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                        "hadoop2:2181,hadoop3:2181");
        // 构建Admin对象
        HBaseAdmin admin = new HBaseAdmin(conf);

        // 创建表的描述以创建表
        TableName name = TableName.valueOf("testTab");
        HTableDescriptor desc = new HTableDescriptor(name);

        // 创建列族
        HColumnDescriptor cf1 = new HColumnDescriptor("cf1");
        HColumnDescriptor cf2 = new HColumnDescriptor("cf2");
        desc.addFamily(cf1);
        desc.addFamily(cf2);

        // 创建表
        admin.createTable(desc);

        // 关闭连接
        admin.close();
    }

}
```

在HBase中，可以看到创建的表：

```text
hbase(main):003:0> list
TABLE
tab1
testTab
2 row(s) in 0.0120 seconds

=> ["tab1", "testTab"]

hbase(main):004:0> desc 'testTab'
Table testTab is ENABLED
testTab
COLUMN FAMILIES DESCRIPTION
{NAME => 'cf1', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER',
COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}
{NAME => 'cf2', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER',
COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}
2 row(s) in 0.1670 seconds
```

往表里头写入数据，使用HTable对象：

```java
package cn.lazycat.bdd.hbase;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.client.Put;

import java.io.IOException;

public class HBasePut {

    public static void main(String[] args) throws IOException {

        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                "hadoop2:2181,hadoop3:2181");
        // 连接HBase数据表
        HTable hTable = new HTable(conf, "testTab");

        // 新建Put，需要传入行键名称
        Put put = new Put("rk1".getBytes());
        // 增加数据，需要列族，列，具体数据
        put.add("cf1".getBytes(), "c1".getBytes(), "value".getBytes());
        // 注意以上全部传入字节数据

        // 向表写入数据
        hTable.put(put);

        // 关闭连接
        hTable.close();
    }

}
```

可以看到结果：

```text
hbase(main):005:0> scan 'testTab'
ROW                                      COLUMN+CELL
rk1                                     column=cf1:c1, timestamp=1523592364481, value=value
1 row(s) in 0.0990 seconds
```

查询数据的操作是类似的：

```java
package cn.lazycat.bdd.hbase;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.Get;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.client.Result;

import java.io.IOException;

public class HBaseGet {

    public static void main(String[] args) throws IOException {

        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                "hadoop2:2181,hadoop3:2181");
        // 连接HBase数据表
        HTable hTable = new HTable(conf, "testTab");

        // 新建Get对象，需要指定行键
        Get get = new Get("rk1".getBytes());
        // 指定列族和列
        get.addColumn("cf1".getBytes(), "c1".getBytes());

        // 查询数据，返回Result对象，因为可能包含多个结果
        Result res = hTable.get(get);
        // 取出结果数据，需要指定列和列族
        byte[] val = res.getValue("cf1".getBytes(), "c1".getBytes());
        // 转换为String，输出
        System.out.println("res = " + new String(val));

        // 关闭连接
        hTable.close();
    }

}
```

在控制台输出：

```text
res = value
```

删除数据：

```java
package cn.lazycat.bdd.hbase;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.Delete;
import org.apache.hadoop.hbase.client.HTable;

import java.io.IOException;

public class HBaseDelete {

    public static void main(String[] args) throws IOException {

        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                "hadoop2:2181,hadoop3:2181");
        // 连接HBase数据表
        HTable hTable = new HTable(conf, "testTab");

        // 新建Delete对象
        Delete delete = new Delete("rk1".getBytes());
        delete.deleteColumn("cf1".getBytes(), "c1".getBytes());

        hTable.delete(delete);

        // 关闭连接
        hTable.close();
    }

}
```

到HBase中，发现数据被删除了：

```text
hbase(main):006:0> scan 'testTab'
ROW                                      COLUMN+CELL
0 row(s) in 0.0210 seconds
```

最后，是删除整个表，使用的是HAdmin对象：

```java
package cn.lazycat.bdd.hbase;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.HBaseAdmin;

import java.io.IOException;

public class HBaseDrop {

    public static void main(String[] args) throws IOException {
        // 取得Conf
        Configuration conf = HBaseConfiguration.create();
        // 配置zk地址
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                "hadoop2:2181,hadoop3:2181");
        // 构建Admin对象
        HBaseAdmin admin = new HBaseAdmin(conf);

        // 禁用表
        admin.disableTable("testTab");

        // 删除表
        admin.deleteTable("testTab");

        // 关闭连接
        admin.close();
    }

}
```

结果，表被删除了：

```text
hbase(main):007:0> list
TABLE
tab1
1 row(s) in 0.0120 seconds

=> ["tab1"]
```
