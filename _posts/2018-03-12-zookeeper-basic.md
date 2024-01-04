---
layout:       post
title:        "Zookeeper基础"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Zookeeper
---

Zookeeper(以下简称ZK), 动物管理员。是一个分布式应用程序的协调服务框架，是Hadoop的一个重要组成组件。

分布式应用需要解决的问题：

- 数据一致性
- 统一的命名服务
- 配置管理
- 分布式锁
- 集群管理

## ZK安装

参见官网教程...(需要安装在Linux系统下)

## ZK指令和数据结构

ZK有一个最开始的节点(/)。ZK的节点叫做znode节点，每个znode节点都可以存储数据，每个znode节点都可以创建自己的子节点。

znode和ZK根节点组成了一个znode树。znode树维系在内存中，查询起来是很快的。

每个znode都有一个名称，并且这个名称是唯一的。

要想运行ZK的指令，需要先启动ZKServer，然后启动ZKClient。

在ZK的bin路径下，执行下面两个命令：

```bash
./zkServer.sh start
./zkCli.sh
```

启动ZK客户端后可以进入ZK的命令行模式，有如下常见的命令：

- ls: 查看节点，例如：ls /
- create znode data：创建节点，例如：create /zk01 hello
  - 不加参数表示创建普通持久节点
  - 加上-e参数表示创建普通临时节点，这样的节点在连接断开后将会消失。利用临时节点可以检查集群节点有没有挂掉。注意临时节点不能拥有子节点。
  - 加上-s表示创建一个顺序持久节点，这样的节点在不同的服务器抢占时会产生一个递增的顺序号。利用它可以实现分布式锁。
  - 加上-s -e就表示创建临时顺序持久节点。
- get znode：查看节点数据，含有以下信息：
  - DATA: 具体数据
  - cZxid: 节点创建时的zxid
  - ctime: 创建节点的时间戳
  - mZxid: 节点最新一次更新发生的zxid
  - mtime: 修改此节点数据的最新时间戳
  - cversion: 子节点的更新次数
  - dataVsersion: 数据版本号，每次数据发生修改，递增1
  - aclVersion: 节点ACL（授权信息）更新次数
  - ephemeralOwner: 如果节点为临时的，表示和该节点绑定的session id。如果不是则为0。
  - dataLength: 数据大小
  - numChildren: 子节点个数
- set znode data: 修改节点数据
- delete znode: 删除一个节点。注意如果节点有子节点则禁止删除。
- quit: 退出ZK客户端

对于ZK集群中每个节点来说，znode树都是一致的，命名也是完全一致的。具体在后面会有介绍。S

## ZK API

Zookeeper是由Java实现的分布式框架。下面我们来看如果通过Java程序来操作znode。

### 基本方法

ZooKeeper是一个类，构造方法参数如下：

- connectString: 包含连接信息的字符串，格式是：zk服务IP地址:端口号。zk默认服务监听端口是2181。
- sessionTimeout: 表示会话的超时时间，单位是ms。
- watcher: 观测者对象

在默认情况下，创建时采用的是非阻塞的模式。

以下代码可以检测是否连接上zkServer：

```java
public class TestConnect {

    public static void main(String[] args) throws IOException {

        ZooKeeper zk = new ZooKeeper("192.168.117.21:2181", 3000,
            (watchedEvent) -> System.out.println("连接成功！")
        );

        while (true);

    }

}
```

加上while(true)是为了防止main线程提前结束导致"连接成功"无法被及时打印出来。

在连接的时候可能产生java.net.ConnectException。这是因为在服务器上没有打开zkServer。

我们可以使用闭锁的技术来保证Zookeeper连接成功再执行接下来的代码：

```java
public class TestConnect {

    public static void main(String[] args) throws IOException,
            InterruptedException {

        final CountDownLatch count = new CountDownLatch(1);

        ZooKeeper zk = new ZooKeeper("192.168.117.21:2181", 3000,
            (watchedEvent) -> {
                System.out.println("连接成功！");
                count.countDown();
            }
        );

        count.await();

        // 执行到此处，连接一定是成功的！
    }

}
```

利用zk对象可以做上面介绍的在终端中的操作。

创建znode的API如下：

- String create(String path, byte data[], List&lt;ACL&gt; acl, CreateMode createMode);

其参数的含义如下：

- path: 表示要创建znode的名称（绝对路径）
- data: 表示节点保存的数据
- acl: 表示节点的权限，在ZooDefs.Ids下有定义，例如OPEN_ACL_UNSAFE就表示所有人都可以对节点进行CRUD。
- createMode: 表示创建的节点模式，在CreateMode下有如下选择：
  - PERSISTENT: 持久普通节点
  - EPHEMERAL: 临时普通节点
  - PERSISTENT_SEQUENTIAL: 持久序列节点
  - EPHEMERAL_SEQUENTIAL: 临时序列节点

创建节点的代码：

```java
zk.create("/zk01/node1", "some data...".getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

zk.create("/zk01/node2", "other data...".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
```

此时就可以在客户端看到创建的节点：

```bash
[zk: localhost:2181(CONNECTED) 9] ls /zk01
[node2, node1]
[zk: localhost:2181(CONNECTED) 10] get /zk01/node1
some data...
cZxid = 0x1f
ctime = Wed Jan 17 08:32:41 CST 2018
mZxid = 0x1f
mtime = Wed Jan 17 08:32:41 CST 2018
pZxid = 0x1f
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 12
numChildren = 0
[zk: localhost:2181(CONNECTED) 11] get /zk01/node2
other data...
cZxid = 0x20
ctime = Wed Jan 17 08:32:41 CST 2018
mZxid = 0x20
mtime = Wed Jan 17 08:32:41 CST 2018
pZxid = 0x20
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 13
numChildren = 0
```

如果节点已经存在了，会抛出org.apache.zookeeper.KeeperException$NodeExistsException这个异常。

对于create()方法，还有一个重载版本：

- void create(path, data, acl, mode, StringCallBack scb, Object ctx);

多的两个参数描述如下：

- scb: 节点创建成功之后调用接口中的processResult(int rc, String path, Object ctx, String name)。方法参数描述如下：
  - rc: 0表示创建成功，-110表示创建失败
  - path: 创建后的节点路径，这个path是由create调用的时候传进来的。
  - ctx: 从外部传入的信息
  - name: 节点名称。创建成功，和path相同，失败则为null。
- ctx: 传入的信息

这个重载create的使用示例如下：

```java
zk.create("/zk03", "haha".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.EPHEMERAL, (rc, path, ctx, name) -> {

    if (rc == 0) {
        System.out.println("节点创建成功！");
    }


}, 110);
```

取得节点详细信息（get）API定义如下：

- byte[] getData(String path, Watcher watcher, Stat stat);

它对应的参数描述如下：

- path: 节点的全路径
- watcher：观测者对象
- stat: 表示get后节点的状态

stat实际上是除了节点数据以外的节点相关信息。可以通过各种get方法获取这些信息。

使用getData来获取/zk01/node1的示例如下：

```java
// getData():
Stat stat = new Stat();
byte data[] = zk.getData("/zk01/node1", null, stat);
System.out.println("data: " + new String(data));
System.out.println("stat: " + stat);
```

对应于getData，还有setData函数：

- Stat setData(String path, byte data[], int version);

参数描述：

- path: 要修改节点的路径
- data: 修改的数据
- version: 表示节点的版本号

特别注意version。它必须和节点原来的version一样，否则会抛出KeeperException$BadVersionException。如果设置为-1则表示不管版本是什么都强制执行。

这个方法会返回节点的状态。

所以我们可以这么使用getData():

```java
// setData():
zk.setData("/zk01/node1", "aaaaaa".getBytes(), -1);
```

ZK还提供了获取某个节点所有子节点的方法：

- List&lt;String&gt; getChildren(String path, Watcher watcher, [Stat stat]);

它返回的是子节点的名称。注意stat保存的是父节点的信息。

例如通过Java查看/zk01的子节点：

```console
[zk: localhost:2181(CONNECTED) 14] ls /zk01
[node2, node1]
```

则使用代码：

```java
// getChildren():
List<String> children = zk.getChildren("/zk01", null);
for (String child : children) {
    System.out.println("子节点：" + child);
}
```

终端输出：

```console
子节点：node2
子节点：node1
```

删除节点使用：

- void delete(String path, int version);

同样，如果不确定版本，version就传-1。

### 监听节点变动

我们发现几乎所有改变节点的额方法都可以传一个Watcher对象。这实际上就是一个监听器，它可以实时监听节点的变动并产生特定的行为。

这个接口的process(WatcherEvent)就是在每次节点变动的时候会调用的方法。watcherEvent记录了事务的类型，我们可以通过getType()方法来取出事务类型。

例如，我们在使用get的时候可以这么写：

```java
Stat stat = new Stat();
byte data[] = zk.getData("/zk01/node1", (watcherEvent) -> {

    if (watcherEvent.getType() ==
            Watcher.Event.EventType.NodeDataChanged) {
        System.out.println("节点被修改了！");
    }

}, stat);
while (true);
```

死循环用于一直保持main线程的状态从而一直监听节点的变动。

这样，在zkClient中输入命令：

```console
[zk: localhost:2181(CONNECTED) 17] set /zk01/node1 new
```

我们会发现Java终端打印出了"节点被修改了！"。

那么我们再修改一次数据，会发现Java终端没有再打印了。这是watcher的特性，它只会监听一次节点的事件。当事件发生完成之后，它就会被销毁。

如果要实现永久监听，可以修改代码：

```java
Stat stat = new Stat();
while (true) {
    final CountDownLatch cdl = new CountDownLatch(1);
    byte data[] = zk.getData("/zk01/node1", (watcherEvent) -> {

        if (watcherEvent.getType() ==
                Watcher.Event.EventType.NodeDataChanged) {
            System.out.println("节点被修改了！");
            cdl.countDown();
        }

    }, stat);
    cdl.await();
}
```

采用这种轮询+闭锁的方式，就可以实现对节点某个事件的持久化监听。

我们还可以对某个节点的所有子节点进行监听：

```java
zk.getChildren("/zk01", (watcherEvent) -> {
    System.out.println("zk01的子节点被改变啦！");
});
```

这个子节点监听只会监听子节点的创建和删除，对于数据的修改则不会监听。

## ZooKeeper集群

上面的操作都是在单机情况下进行的，下面演示如何真正地创建Zookeeper集群。

conf/zoo.cfg是ZK的核心配置文件，集群的配置就是在此处完成的。

首先来看zoo.cfg的配置说明：

- ticktime: 心跳间隔周期，毫秒。
- initLimit: 初始连接超时的预值。指的是follower初始连接leader时会等待initLimit * ticktime的时间，如果时间过了还没有连接上，则放弃连接。
- syncLimit: 指的是follower和leader做数据交互的超时时间，也是syncLimit * ticktime。
- dataDir: 指的是zookeeper的持久化目录。
- clientPort: 指的是zookeeper服务使用的端口号。

在zoo.cfg中需要配置各个节点的信息：

- server.#=ip:port1:port2

\#表示这个节点的选举号。ip表示节点的ip地址，port1表示选举端口，port2表示原子广播的端口。

例如，有三个节点的集群可以在后面这样配置：

```bash
server.1=192.168.117.22:2888:3888
server.2=192.168.117.23:2888:3888
server.3=192.168.117.24:2888:3888
```

随后需要在dataDir指定的目录中创建一个myid文件，保存当前机器的选举号。

配置完以后就可以启动zookeeper服务了。

ZK集群中，有一个节点是作为leader的，它是用来管理整个分布式资源的。其余节点作为follower，它们是被leader管理的。

通过./zkServer status可以查看当前节点是leader还是follower。区别在于Mode上。例如，下面就是一个leader节点：

```bash
[root@localhost bin]# ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/soft/zookeeper-3.4.7/bin/../conf/zoo.cfg
Mode: leader
```

### 选举机制

znode节点的状态信息中包含czxid和mzxid，ZK状态的每一次改变，都对应一个递增的Transaction id，该id称为zxid。由于该zxid递增的性质，如果zxid1小于zxid2，那么zxid1肯定先于zxid2发生。创建节点、更新节点数据、删除节点，都会导致zxid加一。

在ZK选举新的leader的时候。会遵循以下的选举原则：

- 首先比较zxid，最大的作为leader。
- 如果zxid比较不出来，则比较myid(选举id)，最大的当leader。
- 选举的前提是满足过半同意

因为过半同意的机制，所以ZK集群做单数节点数，选举出leader的概率更大。

ZK的选举分为两个阶段：

- 数据回复阶段：当ZK服务器启动时，会先从本地磁盘找到本机的最大事务id
- 选举阶段
  - ZK服务器会把zxid和myid提交从而进行选举
  - ZK维护一个逻辑时钟值，确保每个节点在同一轮选举中
  - ZK服务器一共有四种状态：
    - Looking：选举阶段
    - Following：小弟
    - Leading：领导
    - Observering：观察者。这种节点不会参与选举，适合大集群情况。

在Leader选取出来之后，其身上肯定是有最新数据的（有最大的事务id），所以首先做的事就是原子广播（通过原子广播端口），让follower更新自己的数据。这是为了确保数据的一致性，防止因为leader的宕机产生的数据不一致。

对于每次事务更新，leader会通过原子广播征询其它follower，只有过半同意才会真正地执行事务。而且这样的事务执行是在所有节点上都会执行的。

一般来说，在投票的时候，节点都会投赞同票。除非网络异常或节点宕机才会让节点投反对票。

那么也就是说，在一个集群中如果宕机的机器太多（超过半数），那么这个集群就无法工作了（任何事务投票都不会通过），直到运维人员修复至少一半的节点。

所以我们在一个follower节点执行一个create操作，那么在其它所有节点执行ls /操作会发现这个数据已经全部被创建了。

在节点3(follower)操作：

```console
[zk: localhost:2181(CONNECTED) 1] create /zk01 nihao
Created /zk01
```

在节点2(leader)查看：

```console
[zk: localhost:2181(CONNECTED) 0] ls /
[zk01, zookeeper]
```

要是集群节点正常，理论上对一个znode的修改会导致集群所有节点的zxid递增。所以ZK为什么要在选举的时候优先选择zxid高的，就是为了保证即使在异常发生以后，数据也能尽可能保证最新化，一致化。
