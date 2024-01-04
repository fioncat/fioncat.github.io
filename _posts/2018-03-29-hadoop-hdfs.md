---
layout:       post
title:        "Hadoop笔记二：HDFS"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Hadoop
---

HDFS是Hadoop为了储存海量数据而使用的一种分布式文件系统。这种文件系统是运作于多个机器之上的。

HDFS为了保证数据储存的可靠和读取性能，会把保存的数据进行切块后进行复制并且储存在集群的多个节点中。

HDFS存在名字节点NameNode和数据节点DataNode：

- NameNode：储存元数据信息，也就是具体文件，block，datanode之间的映射关系。数据保存在内存和磁盘中。这是HDFS最重要的节点，通过NameNode才能读取文件。
- DataNode：储存block内容，维护block id到文件的映射关系，这样可以加快读取文件时的效率。数据储存在磁盘中。

HDFS优点：

- 支持储存超级大文件，可以支持储存PB级别大小的文件(如此大文件很难储存在一块硬盘，所以普通的文件系统根本储存不了)，对于企业级别的应用，数据节点可能有上千个。
- 可以检测和快速应对硬件故障。这个优点特别重要，多节点集群有很高的故障率。所以HDFS对于故障检测和恢复的支持是比较好的。
- 流式数据访问，由于HDFS处理的数据比较大，所以HDFS以流来访问数据集，数据的吞吐量较大。
- 简化的一致性模型，HDFS操作文件时，一般是一次写入，多次读取。文件一旦创建，几乎很少修改。修改的效率非常低。
- 高容错，数据会被保存多个副本，副本丢失允许恢复。
- 可以通过扩展廉价机数量来提高效率。

缺点

- Hadoop以数据访问的延时为代价换取来换取吞吐量。所以不适合低延迟应用。
- HDFS不适合储存大量的小文件。大量小文件会使得NameNode非常大，小内存NameNode甚至不可以保存过多的文件。
- 不支持修改和追加写入(2.x支持追加)
- 对事务的支持不像关系型数据库强大。

## HDFS 基本概念

### Block

Block，数据块。HDFS一般储存超大规模的文件，HDFS一般按照一定标准把文件拆分成多块(block)，对于1.x版本，一个Block大小为63M，2.x为128M。

Block可以保存在不同的磁盘上。HDFS在管理文件的时候实际上就是在管理block的分布。

在HDFS中，一个block通常会被复制3份。

如果一个文件的长度小于block的大小，那么这个文件并不会占用整个Block的空间大小。

### NameNode

NameNode的作用是维护HDFS的元数据，包括文件是什么，文件被分成了几个Block，Block被储存在了那里。

NameNode将元数据储存在内存和磁盘中。内存储存的是实时数据，磁盘为数据的映像作为持久化储存(方便NameNode宕机时的重启)。

磁盘中存储3种文件：

- fsimage: 元数据的镜像文件。储存某NameNode元数据的信息。但不是实时同步内存的数据。
- edits: 操作日志文件，这个日志是实时的。
- fstime: 保存最近一次checkpoint时间。

当有写请求的时候，NameNode会首先把写操作的日志写到edits中，成功后修改内存，并向客户端返回结果(并不会马上改变fsImage)

根据edits的日志真正操作fsImage更新磁盘数据会在一定特定条件下发生。这种操作被叫做“合并”，也叫checkpoint，在合并发生后，合并时间会被记录在fstime中。

合并的操作并不是由NameNode完成的(NameNode压力不能太大)，是由SecondaryNameNode完成的。

SecondaryNameNode并不是NameNode的热备份，而是专门协助NameNode进行元数据合并的(fsimage和edits文件)，同时，它能提供冷备份，因为它也保存了部分NameNode数据。但是在极端的情况下，NameNode如果挂掉，SecondaryNameNode也许不能提供完整的数据恢复。

我们可以控制SecondaryNameNode的合并间隔，可以根据fs.checkpoint.period来设置，默认是3600秒。

配置项fs.checkpoint.size规定了edits文件的最大大小，当达到或者超过这个大小，也会进行一次合并(即使period没有到)。

在创建文件的时候，整个流程如下(NN&SNN架构)：

- NameNode受到请求，会先检查edits的大小和有没有到checkpoint。如果edits太大或者已经到了checkpoint，会创建一个新的edits.new文件并将请求写入。
- NameNode会把原来的edits文件和fsimage文件通过HTTP GET发送给SecondaryNameNode。
- SecondaryNameNode根据edits和fsimage进行合并，将合并后的fsimage发送给NameNode。
- NameNode使用接收到的新的fsimage替换原来的fsimage，并将新的edits.new文件替换掉原来的。

这样做的好处是，在SecondaryNameNode进行合并操作的期间，新的请求会被写到edits.new文件中。合并完成之后edits.new替换原来的edits。这样一来合并对于用户来说就是透明的了，合并操作不会带来阻塞。

合并的另外一个好处是，SecondaryNameNode从一定程度上能够得到NameNode的fsimage和edits文件，这样在NameNode挂掉之后，SecondaryNameNode能做一些恢复(但是在合并期间的请求会丢失)。这就是所谓的“冷备份”。冷备份的恢复时间会比热备份要高得多。

考虑到效率和安全，NameNode和SecondaryNameNode不能在同一台机器。

### DataNode

在Hadoop集群中，DataNode是负责储存数据的。数据以Block的形式储存在DataNode中。

DataNode会不断地向NameNode发送心跳报告，告知NameNode自己是存活的。

初始化时，每个DataNode会将当前储存的Block告知NameNode。随后DataNode就会使用心跳报告机制和NameNode保持联系(一般是3秒一次)。

DateNode也保存了一份元数据信息，在之后的工作中，它会不断接收来自NameNode的指令，从而更新元数据信息并且由指令对Block进行读取，创建，移动或删除。

如果NameNode10分钟没有受到来自DataNode的心跳，就认为这个DataNode挂掉了，并且将数据的Block复制到其它DataNode上。

所有的Block都会默认保存3份副本(真实集群)，副本会被保存到不同的DataNode上。这样即使某个DataNode挂了，数据也不会丢失。

放置副本的策略(在机架策略打开的情况下)：

- 第一个副本：如果上传文件的服务器是DataNode，就直接放置在上传文件的服务器上。如果是外部的客户端上传的文件，会随机选择一个DataNode(磁盘不忙，CPU占用率不高的机器)
- 第二个副本：放置在第一个副本不同的机架上。
- 第三个副本：放置在第二个副本相同的机架上(机架内通信速度比跨机架快)。
- 更多副本：随机选择节点。

那么如何知道两个机器是不是在一个机架上呢？这就需要用到机架感知策略了。这里不再详细介绍，有兴趣可以查阅资料。

## HDFS 指令

在Linux下可以使用Hadoop指令操作HDFS，这些命令和Linux的bash命令很相似：

- hadoop fs -l DIR: 查看DIR路径。
- hadoop fs -mkdir DIR: 创建路径。
- hadoop fs -lsr DIR 或 hadoop fs -ls -R DIR(规范): 递归查询目录
- hadoop fs -put current PATH: 将本地的文件(current)复制到HDFS中。
- hadoop fs -get PARH current: 将HDFS中的文件下载到本地(current)。
- hadoop fs -rm PATH: 删除HDFS中的文件
- hadoop fs -rmdir PATH: 删除空目录
- hadoop fs -rmr PATH 或 hadoop fs -rm -r PATH(规范): 递归删除目录
- hadoop fs -cat PATH: 查看文件
- hadoop fs -mv PATH1 PATH2: 移动
- hadoop fs -touchz PATH: 创建一个空的文件或修改文件修改日期
- hadoop fsck PATH: 汇报目录的健康情况

## HDFS 写流程

在HDFS中写入数据的过程如下：

- 使用HDFS提供的客户端开发库(Client)，向远程NameNode发送RPC请求，NameNode会首先检查要创建的文件是否存在，以及创建者是否有权限进行操作，成功则会为文件创建一个记录，否则让客户端抛出异常。
- NameNode创建记录之后，开发库会把文件切分成多个packets(比block更加细)，并在内部以数据队列的方式管理，并且向NameNode申请blocks。NameNode会返回用来储存数据的合适的DataNodes列表，列表的大小(决定了一个block会有几个replicas)由NameNode中的replication指定。
- Client会根据DataNodes列表建立一个pipeline(管道)，这个管道连接了所有要储存数据的DataNode。packet会通过pipeline经过第一个DataNode，该DataNode储存数据之后，再将其传递给在此pipeline中的下一个DataNode，整个写数据的方式呈现流水形式。
- 最后一个DataNode写完数据之后，返回一个ack packet给Client，当Client收到DataNode的ack packet之后，会从packets队列中移除刚刚增加的packet。
- 当所有数据写入完毕之后，Client会通知NameNode关闭文件。
- 如果在Client向DataNode传输数据的途中，DataNode发生了故障，那么当前pipeline会被关闭，出现故障的DataNode会被从pipeline移除，剩余的block继续传输，同时NameNode会分配一个新的DataNode。

## HDFS 读流程

HDFS读取数据的流程如下：

- Client远程通过RPC向NameNode发送读请求。
- NameNode会视情况返回文件的部分或者全部block列表。对于每个block，NameNode同时会返回它replicas的位置。
- Client会选取离Client最近的DataNode开始读取，如果客户端本身就是DataNode并且block就在自己身上(这种情况经常出现)，那么直接在本地读取。
- 读取完一个block之后，关闭和这个block的连接，并继续寻找下一个最佳的block读取。
- 当读取完NameNode发过来列表中的所有blocks之后，Client发现文件还是没有读取完毕，就会再向NameNode申请文件剩下来的blocks列表，继续读取。
- 如果读取DataNode时出现错误，客户端会通知NameNode，再从下一个拥有这个block拷贝(replicas)的DataNode继续读取。
- 读取完毕之后，DataNode会负责告知NameNode读取完毕，关闭文件。

这里有一个很有意思的机制，那就是在读取的时候，如果Client发现某个DataNode有故障，它不会只是简单地跳过这个DataNode去读取replicas，而是会把这一情况告知NameNode。

NameNode受到这样的报告之后，就会确保以后再有数据读取的时候，不会再返回这个有故障的DataNode的位置了(会用它的replicas取代)。可以防止Client重复读取一个故障节点。

Client同时会检查接收到的数据的校验和，如果发现数据损坏，也会通知NameNode。

这个设计要求Client在读取数据的过程中始终保持和NameNode的联系。并且NameNode会指引Client读取最好的DataNode，这有利于集群的整体性能。

## HDFS 删除流程

在删除某个文件的时候，会先在NameNode上执行Name的删除。

注意这时只是标记要删除的数据块，而没有真正通知DataNode去删除这个文件的blocks。

在保存着这些blocks的DataNode向NameNode发送心跳的时候，NameNode才会给DataNode发送指令，进而完全把block删除。

所以在执行完delete后，要过一段时间，数据才会被完全删除。

## 安全模式

在重新启动HDFS之后，会立即进入安全模式，此时不能操作HDFS的文件，只能查看文件名。

随后NameNode启动之后，需要载入fsimage并写入到内存，同时会执行一遍edits的各种操作，以保证数据是尽量新的。

一旦在内存中建立好fs映射，会创建一个新的fsimage文件，和一个空的edits文件。

这是一个归并过程，这个“重启归并”是NameNode亲自做的，SecondaryNameNode没有参与。

在NameNode进行归并的时候，FS对于Client是只读的。

在此阶段NameNode还会收集来自各个DataNode的报告，当所有blocks达到最小的replicas数以上时，认为这个block是安全的。

在一定比例的block被认定为安全之后，再等待若干时间，安全模式结束。

NameNode检测到replicas数量不足的block时，会再进行一次replication。

以下是HDFS提供的和安全模式相关的命令：

- hadoop dfsadmin -safemode get: 查看安全模式的开关状态
- hadoop dfsadmin -safemode leave: 离开安全模式(不建议轻易尝试)
- hadoop dfsadmin -safemode enter: 进入安全模式
- hadoop dfsadmin -report: 查看存活的DataNode信息

## JAVA API

HDFS是由Java实现的，并且整个Hadoop都是以Java为主的，所以HDFS提供了Java API以方便用户开发客户端程序远程操作HDFS。

### FileSystem 读流程

使用Hadoop提供的FileSystem类对HDFS进行读的过程和通过终端命令略有不同：

- 客户端通过调用FileSystem类的对象(子类为DistributedFileSystem)的open()来获取希望打开的文件。对于HDFS来说，这个对象是分布式文件系统的一个实例。
- DistributedFileSyste通过RPC来调用NameNode，以确定文件的开头部分的block的位置。对于每一block，NameNode返回具有该block的DataNode的地址，这些DataNode会根据网络集群的拓扑位置进行排序。如果该Client本身就是一个DataNode，那么会优先从本机读取。
- DistributeFileSystem返回一个FSDDataInputStream对象被Client读取数据，FSDDataInputStream转而包装了一个DFSInputStream与最近的DataNode连接。
- Client对这个输入流调用read()方法，数据就会从DataNode返回给Client。
- 当读到block的末端时，DFSInputStream会关闭和DataNo的联系，然后为下一个block找DataNode，继续读取。
- Client只需要读一个连续的流，对于block的分块读取是透明的。

### FileSystem 写流程

- Client调用DistributFileSystem的create()方法来创建文件。
- DistributeFileSystem使用RPC调用NameNode。
- NameNode会检查，确保这个文件不会已经存在了，并且确保这个Client有创建这个文件的许可。如果一切正常，NameNode会创建新的文件记录，否则，会向Client抛异常。
- 创建文件记录后，NameNode会返回一个文件系统的输出流，让Client开始写入数据。我们通过DFSOutputStream向DataNo的写数据。
- 在Client写入数据的时候，DFSOutputStream将它们分成一个一个packet，并创建一个packet的发送队列和一个确认队列。
- 在发送数据之后，发送的packet会被从发送队列移除并放到确认队列中。
- 在DataNode确认保存数据后，会返回一个信息，这时就把保存成功的packet从确认队列删除。
- 如果DataNode坏掉了，就会把pipeline删掉，把确认队列的packet放回发送队列的头部，通知NameNode，接着重新建立pipeline发送。这是为了确保在产生故障时不会产生丢包的情况。

### API

FileSystem类是整个操作的核心类，这是一个抽象类。通过静态方法get()可以取得对象：

```java
public static FileSystem get(URI uri, Configuration conf, String user);
public static FileSystem get(URI uri, Configuration conf);
```

Uri表示访问的服务器的位置，格式是"hdfs://ip:port"。

注意这个方法返回的是分布式文件系统，Hadoop还提供了其它文件系统，这里不再叙述。

需要用到配置类Configuration，这个类可以通过默认构造创造。它可以对HDFS进行一些配置，利用set()方法完成：

```java
public void set(String name, String value);
```

这个方法可以设置一些Hadoop的参数，如果不设置，那么就使用默认的参数。

在使用FileSystem之后，记得调用close()方法关闭和FS的连接。

我们可以通过fs对象来进行一些简单的文件操作。例如，创建目录：

```java
public boolean mkdirs(Path path);
```

Path对象是Hadoop提供的，通过构造方法可以把路径传进去。

下面的代码是创建目录的示例：

```java
public class TestMkDir {

    public static void main(String[] args) throws URISyntaxException, IOException {

        Configuration conf = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI("hdfs://192.168.117.51:9000"), conf);
        System.out.println(fileSystem);
        fileSystem.mkdirs(new Path("/test01"));
        fileSystem.close();
    }

}
```

随后在Linux输入：hadoop fs -ls /。就可以看到创建的test01目录了。

我们通过FileSystem的create()方法，可以创建一个新的文件并取得写入数据的FSDataOutputStream流：

```java
public FSDataOutputStream create(Path f);
```

这会在NameNode申请一个新的文件并且把写如这个文件的流传给客户端。

那么，接下来的操作就和操作普通流一模一样了。下面是把一个本地文件写入HDFS的例子(IOUtils可以直接把流的内容复制)：

```java
public class TestUpload {

    public static void main(String[] args) throws URISyntaxException, IOException {

        FSDataOutputStream out = null;
        InputStream in = null;
        FileSystem fs = null;
        try {

            Configuration conf = new Configuration();
            fs = FileSystem.get(new URI("hdfs://192.168.117.51:9000"), conf);

            out = fs.create(new Path("/test01/test.txt"));
            in = new FileInputStream("test.txt");

            IOUtils.copyBytes(in, out, 2048);

        } finally {

            if (in != null) {
                in.close();
            }

            if (out != null) {
                out.close();
            }

            if (fs != null) {
                fs.close();
            }
        }

    }

}
```

默认情况下，会对这个文件的block保存3个副本，我们可以通过"dfs.replication"这个配置项修改这一行为。在程序中需要调用Configuration的set()方法：

```java
conf.set("dfs.replication", "1");
```

这样就不会产生副本了（适合于伪分布式情况）

至于读取操作，则和写很像，FileSystem可以通过open()方法来取得要读取文件的FSDataInputStream对象：

```java
public FSDataInputStream open(Path f);
```

下面是读取文件的示例代码：

```java
public class TestDownload {

    public static void main(String[] args) throws URISyntaxException, IOException {

        FileSystem fs = null;
        FSDataInputStream in = null;

        try {

            Configuration conf = new Configuration();
            fs = FileSystem.get(new URI("hdfs://192.168.117.51:9000"), conf);
            // InputStream 的后代类
            in = fs.open(new Path("/test01/test.txt"));

            // 将信息输出在终端
            OutputStream out = System.out;

            IOUtils.copyBytes(in, out, 2048);

        } finally {

            if (in != null) {
                in.close();
            }

            if (fs != null) {
                fs.close();
            }
        }
    }

}
```

可见我们之前介绍的所有关于block，packet的操作在这里都是完全透明的。我们只需要像处理一般的流一样去操作就可以了。

我们甚至可以通过FileSystem来查看某个文件的block的位置：

```java
public BlockLocation[] getFileBlockLocations(Path p, long start, long end);
```

参数说明：

- p: 表示要查看哪个文件的block位置
- start: 从哪个位置开始查看，0开始
- end: 查看到哪里，要查看全部的话可以使用Long.MAX_VALUE。

下面的代码演示了如何查看HDFS中某个文件(这里是"/test01/testData.tar.gz"，这个文件大约有200M)：

```java
public class TestBlock {

    public static void main(String[] args) throws URISyntaxException, IOException {
        Configuration conf = new Configuration();
        FileSystem fs = null;
        try {

            fs = FileSystem.get(new URI("hdfs://192.168.117.51:9000"), conf);

            BlockLocation[] locations = fs.getFileBlockLocations(
                    new Path("/test01/testData.tar.gz"), 0, Long.MAX_VALUE);

            for (BlockLocation location : locations) {
                System.out.println(location);
            }
        } finally {

            if (fs != null) {
                fs.close();
            }

        }
    }
}
```

在我的机器上，以上代码输出如下：

```text
0,134217728,hadoop1
134217728,76389079,hadoop1
```

可见这个文件有两个块，因为我用的是伪分布式，所以这两个块都储存在一个节点上。

我们还可以查看文件的状态信息，这需要调用FileSystem的以下方法：

```java
public FileStatus[] listStatus(Path f);
```

用起来和查看Block信息很像，下面是演示(只有关键代码)：

```java
fs = FileSystem.get(new URI("hdfs://192.168.117.51:9000"), conf);

FileStatus statuses[] = fs.listStatus(new Path("/test01/test.txt"));
for (FileStatus status : statuses) {
    System.out.println(status);
}
```

控制台输出：

```text
FileStatus{path=hdfs://192.168.117.51:9000/test01/test.txt; isDirectory=false; length=22; replication=3; blocksize=134217728; modification_time=1521715007458; access_time=1521715006307; owner=LazyCat; group=supergroup; permission=rw-r--r--; isSymlink=false}
```

我们可以看到FileStatus类的一些属性，通过相应的getter方法可以取得文件的具体状态。

要想删除文件或目录，则使用FileSystem的delete方法：

```java
public boolean delete(Path f, boolean recursive);
```

注意recursive用于指明是否递归删除文件，如果设置为false并且试图去删除一个非空目录，那么会抛出异常。
