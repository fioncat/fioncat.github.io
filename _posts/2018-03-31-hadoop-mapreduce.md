---
layout:       post
title:        "Hadoop笔记三: MapReduce"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Hadoop
---

MapReduce是一个分布式的计算框架。最初由谷歌的工程师开发，基于GFS的分布式计算框架，主要用于搜索领域解决海量数据的计算问题。

Cutting根据这个框架，设计了基于HDFS的MapReduce框架

MapReduce可以让程序员远离分布式计算编程，不需要考虑任务调度、逻辑切块、位置追溯等问题。他们就可以把精力集中在业务上了。

MapReduce由两个阶段组成：Map和Reduce。用户只需要实现map(inKey,inVal,outKey,outVal)和reduce(ikey,ival,okey,kval)两个函数即可完成分布式计算。

## MapReduce 框架结构

### JobTracker/ResourceManager 工作职能

ResourceManager是hadoop 2.x在引入yarn之后，替换了JobTracker。

- 知道管理哪些机器，即管理哪些NodeManager。
- 通过RPC心跳来检测NodeManager的状态。
- 任务的分配和调度，ResourceManager能够做到细粒度的任务分配，比如某一个任务需要用到多少内存，需要多少计算资源等。

### TaskTracker/NodeManager 工作职能

同样，NodeManager是在Hadoop引入yarn之后使用的。

NodeManager能够处理来自ResourceManager发过来的任务，并进行任务的处理。这里处理的是Map或Reduce任务。

### MapReduce执行步骤

- 首先，要处理的海量数据一般储存在HDFS中。
- 对这海量数据进行读取的时候，会对数据进行划分，划分的基准是行。
- Map作业的输入就是每一行数据，key是这一行开头的偏移量(相对于整个文件的)，value就是这一行的数据。
- Map作业的具体功能是由用户实现的，要求对每一行都产生若干key-value输出(由用户自己定义)
- Map作业结束之后，会对所有Map作业产生的结果进行一次"Aggregation and shuffle"。这会把Map作业的结果进行合并，把相同的key的键值对(value可能不同)合并。例如，key1-value1, key1-value2合并为key1-(value1,value2)。这一过程能够保证所有计算结果的key是唯一的。
- Reduce作业会去获取合并后的key-values。然后经过处理产生新的key-value，具体的处理过程由用户自己定制。一般我们会把values进行统计或者某些计算产生一个value输出。
- Reduce的输出会保存到文件中，这就是MapReduce的最终计算结果。

在Hadoop中，为了方便序列化，用LongWritable表示long类型，Text表示String类型。这些类型都是可以和基本类型互相转换的。

### MapReduce 案例：单词统计

下面以一个最简单的单词统计(相当于MapReduce的hello world)功能来演示MapReduce的API。

我们准备一个简单的文本文件words.txt，我们的目标是统计出这个文件中每个单词出现的个数(每个单词以空格区分)：

```text
hello world
I am a small cat
I love hadoop
I love cat
hello hadoop
hello cat
```

首先，把这个文本文件上传到HDFS(/park/words.txt)：

```text
[root@hadoop1 ~]# hadoop fs -mkdir /park
[root@hadoop1 ~]# hadoop fs -put words.txt /park/words.txt
[root@hadoop1 ~]# hadoop fs -ls /park
Found 1 items
-rw-r--r--   1 root supergroup         77 2018-03-24 12:18 /park/words.txt
[root@hadoop1 ~]# hadoop fs -cat /park/words.txt
hello world
I am a small cat
I love hadoop
I love cat
hello hadoop
hello cat
```

接下来，我们编写Map作业：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

/**
 * Map作业类，需要继承Mapper(注意来自hadoop.mapreduce包)
 * 四个泛型分别表示输入key-value和输出key-value
 * 输入key是long，表示每行开始的偏移量
 * 输入value是Text，表示每行内容
 * 输出key是单词的内容
 * 输出value是单词在这一行出现的次数
 */
public class WordCountMapper
        extends Mapper<LongWritable, Text, Text, LongWritable> {

    /**
     * 用户定制的map方法
     * @param key 输入key
     * @param value 输入value
     * @param context 向外输出的对象
     */
    @Override
    protected void map(LongWritable key, Text value,
                       Context context)
            throws IOException, InterruptedException {

        // 获取读取的一行的文本的内容
        String line = value.toString();

        // 按照空格拆分，取得每一个单词
        String words[] = line.split(" ");

        for (String word : words) {
            // 对于每一个单词，产生一个输出
            context.write(new Text(word), new LongWritable(1));
        }
    }
}
```

Reduce作业：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

/**
 * Reduce作业类
 * 输入是Map作业结果的key-value类型，输入是最终的计算结果
 * 在本次案例，输入是每一个单词出现的标记，Reduce作业将
 * 这些次数做一个求和统计最终产生每个单词和单词出现的次数。
 */
public class WordCountReducer
        extends Reducer<Text, LongWritable, Text, LongWritable> {

    /**
     * reduce作业代码
     * @param key 来自map的key
     * @param values 洗牌后的values
     * @param context 输出结果
     */
    @Override
    protected void reduce(Text key, Iterable<LongWritable> values,
                          Context context)
            throws IOException, InterruptedException {

        long sum = 0;   // 单词出现的次数
        for (LongWritable value : values) {
            sum += value.get();
        }

        // 写出数据
        context.write(key, new LongWritable(sum));
    }
}
```

光有Map和Reduce，我们无法运行代码，我们需要编写一个Job类，表示整个业务处理的启动类。这个类需要一个main方法，作为Hadoop启动业务的入口。

这个类的实现如下：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class WordCountJob {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        // Hadoop 配置
        Configuration conf = new Configuration();
        // Hadoop Job 类
        // 传入conf对象和job的名称(均可缺省)
        Job job = Job.getInstance(conf, "wc");
        // 指定Job的入口类
        job.setJarByClass(WordCountJob.class);
        // 指定Mapper和Reducer
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);
        // 指定Mapper的输出key-value类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(LongWritable.class);
        // 指定Reducer的输出key-value类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);
        // 指明输入的数据(来自HDFS)
        // 这里指定一个目录，Hadoop会读取目录下的所有文件(不会递归)
        FileInputFormat.addInputPath(job,
                new Path("hdfs://192.168.117.51:9000/park"));
        // 指明输出(输出到HDFS)
        // 注意这里不要事先创建，不然会抛出异常
        FileOutputFormat.setOutputPath(job,
                new Path("hdfs://192.168.117.51:9000/out"));
        // 开始执行任务
        job.waitForCompletion(true);
    }

}
```

在Windows下执行的话，记得一定要在hadoop的bin下增加winutils.exe文件。还要把hadoop.dll复制到C:/Windows/System32下(这些文件来自于eclipse的Hadoop插件)，否则会抛出异常。

如果一切正常，执行成功的话，在/out下我们可以看见作业的运行结果：

```text
[root@hadoop1 ~]# hadoop fs -ls /out
Found 2 items
-rw-r--r--   3 LazyCat supergroup          0 2018-03-24 14:24 /out/_SUCCESS
-rw-r--r--   3 LazyCat supergroup         59 2018-03-24 14:24 /out/part-r-00000
```

其中，_SUCCESS表示任务执行成功的标记，这是一个标识文件。partxxx文件就是输出的结果文件。结果可能由多个文件组成。

我们可以查看结果：

```text
[root@hadoop1 ~]# hadoop fs -cat /out/part-r-00000
I   3
a   1
am  1
cat 3
hadoop  2
hello   3
love    2
small   1
world   1
```

可见，输出的结果是: key    value(Reduce作业输出的)。每个键值对占据一行。

我们也可以把代码打包后放到DataNode上执行(通过hadoop命令)：

```text
[root@hadoop1 ~]# hadoop jar wc.jar
```

其中，wc.jar是我打包的项目jar包，注意需要把上面的WordCountJob设置为Main类。

如果成功，输入的日志类似于这样(省略部分日志)：

```text
18/03/24 14:43:44 INFO input.FileInputFormat: Total input paths to process : 1
18/03/24 14:43:45 INFO mapreduce.JobSubmitter: number of splits:1
18/03/24 14:43:45 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1521642825141_0002
18/03/24 14:43:45 INFO impl.YarnClientImpl: Submitted application application_1521642825141_0002
18/03/24 14:43:45 INFO mapreduce.Job: The url to track the job: http://hadoop1:8088/proxy/application_1521642825141_0002/
18/03/24 14:43:45 INFO mapreduce.Job: Running job: job_1521642825141_0002
18/03/24 14:43:58 INFO mapreduce.Job: Job job_1521642825141_0002 running in uber mode : false
18/03/24 14:43:58 INFO mapreduce.Job:  map 0% reduce 0%
18/03/24 14:44:05 INFO mapreduce.Job:  map 100% reduce 0%
18/03/24 14:44:10 INFO mapreduce.Job:  map 100% reduce 100%
18/03/24 14:44:11 INFO mapreduce.Job: Job job_1521642825141_0002 completed successfully
18/03/24 14:44:11 INFO mapreduce.Job: Counters: 49
    File System Counters
        ...
    Job Counters
        ...
    Map-Reduce Framework
        ...
    Shuffle Errors
        ...
    File Input Format Counters
        ...
    File Output Format Counters
        ...
```

作业运行的结果和上面是一样的。

## MapReduce 作业具体流程

要想理解Hadoop中MapReduce作业的执行流程，一定要先搞懂整个作业的三个角色：

- Client Node: 客户端节点，表示执行代码的节点。这个节点可以不在Hadoop集群中或者位于Hadoop集群的某个DataNode上。
- ResourceManager: 整个资源的调度器，来自于YARN框架。作用是给分配各个NodeManager分配任务和资源。在Hadoop集群中一般和NameNode待在一起。
- NodeManager: 相当于任务的执行者。调用jar文件启动Java进程执行任务代码，读取来自共享文件系统(如，HDFS)的任务资源进行处理。

作业的具体流程如下：

### 提交作业

作业是被Client Node提交的，这个我们从上面的源代码就可以看出来了。提交的关键就是submit()方法。我们发现Job实例化了一个JobSubmitter实例，这个实例就是专门用来提交作业的。

如果深入JobSubmitter源代码，会发现它会向ResourceManager请求一个JobId。这个ID是任务的唯一标识。

获取JobID后，JobSubmitter会计算出分片的信息(分片的偏移量)并将作业需要的资源(作业的MapReduce的jar包，配置信息和分片信息)上传到HDFS中。

随后，调用submitJob()方法真正地提交作业。

### 作业初始化

在调用submitJob()之后，ResourceManager会计算(通过yarn调度器scheduler)出一个适合当前任务的计算节点(NodeManager)。然后，通知这个节点的NodeManager启动一个容器(可以想象是存放实现某个功能的代码和实现这个功能需要的资源的空间，每个容器都必须有一个执行主类)。

随后，NodeManager会启动容器中的MRAppMaster进程。这个进程就是专门用来初始化整个Job的。

MRAppMaster进程会从HDFS中下载分片信息(在提交作业时已经由Client Node上传了)。

随后，MRAppMaster会对下载的每一个输入分片创建一个map任务对象和由mapreduce.job.reduces属性指定的多个reduce对象。并且会向ResourceManager请求这些map任务和reduce任务的容器，存放这些任务需要的资源。

任务容器和AppMaster一般在同一个节点，但是如果任务非常大，就会尝试到其它的节点分配任务容器。

这个MRAppMaster进程还可以创建多个薄记对象以保持对作业进度的追踪，并随时返回给Client Node进行日志输出。

### 执行任务

MRAppMaster创建任务容器之后，就告知NodeManager来启动这个容器。

这个容器的主类是YarnChild进程。它启动前会从HDFS上下载作业的配置、jar文件和作业需要的资源。然后，开始启动map任务或者reduce任务。

### 进度和状态更新

在使用yarn运行任务时，会每隔3s通过umbilical接口向AppMaster汇报进度和状态。Client Node每隔一秒钟(通过mapreduce.client.Progressmonitor.pollinterval设置)和AppMaster进行一次通讯以接收任务完成情况，并在前台显示给用户。

### 作业完成

Client Node每隔5秒钟(通过mapreduce.client.completion.pollinterval设置)会调用Job的waitForCompletion()方法来检测作业是否完成。

作业完成后，AppMaster容器和任务容器会被清理。

## MapReduce 序列化

整个集群工作过程中涉及到大量序列化和反序列化操作。Hadoop底层是通过AVRO实现序列化和反序列化的，并且封装AVRO提供了一些便捷的API。

Hadoop对Java基本类型做了一个序列化封装，这些类型都实现了Writable接口。下面是对照表：

Java基本类型 | Writable实现类 | 序列化大小(byte)
-----| --------- | ---|
null | NullWritable.get() |
boolean | BooleanWritable | 1
byte | ByteWritable | 1
Short | ShortWritable | 2
int | IntWritable | 4
int(变长) | VintWritable | 1~5
float | FloatWritable | 4
long | LongWritable | 8
long(变长) | VlongWritable | 1~9
double | DoubleWritable | 8

这些类型的readFields()可以实现序列化，write()可以实现发序列化。

### 序列化案例：统计流量

下面通过一个具体的例子来展示序列化的用法。

这个案例要实现一个统计流量的功能，数据文件的格式如下：

- flow.txt

```text
13877779999 bj zs 2145
13766668888 sh ls 1028
13766668888 sh ls 9987
13877779999 bj zs 5678
13544445555 sz ww 10577
13877779999 sh zs 2145
13766668888 sh ls 9987
```

第一个表示用户的手机号，第二个表示流量产生的位置，第三个表示用户的姓名，最后一个表示产生的流量。

先把数据推到HDFS：

```text
[root@hadoop1 ~]# hadoop fs -put data/flow.txt /park/flow
[root@hadoop1 ~]# hadoop fs -cat /park/flow
13877779999 bj zs 2145
13766668888 sh ls 1028
13766668888 sh ls 9987
13877779999 bj zs 5678
13544445555 sz ww 10577
13877779999 sh zs 2145
13766668888 sh ls 9987
```

我们可以像往常一样，直接编写MapReduce代码(不使用自定义对象)，那么对于Map任务，需要把每行记录的手机号和姓名作为key输出，把流量值作为value输出(忽略地点信息)：

```java
public class FlowMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

    @Override
    protected void map(LongWritable key, Text value,
                       Context context)
            throws IOException, InterruptedException {

        String line = value.toString();

        // 取出三个需要的数据
        String data[] = line.split(" ");
        String phone = data[0];
        String name = data[2];
        int flow = Integer.parseInt(data[3]);

        context.write(new Text(phone + " " + name), new IntWritable(flow));

    }
}
```

在reduce任务中，我们可以把手机号，姓名作为key输出，流量总值作为value输出：

```java
public class FlowReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {

        int sum = 0;
        for (IntWritable val : values) {
            sum += val.get();
        }

        context.write(key, new IntWritable(sum));
    }
}
```

执行这个MapReduce，输出：

```text
[root@hadoop1 ~]# hadoop fs -cat /out/part-r-00000
13544445555 ww  10577
13766668888 ls  21002
13877779999 zs  9968
```

可是，电话，用户名称，流量这三个信息是明显可以通过OOP抽象为一个类的，我们大可把这个类作为输入和输出，这就需要用到Hadoop的序列化机制了。

我们的自定义序列化对象需要实现Writable接口。这需要实现write(DataOutput)和readFields(DataInput)方法，从而实现序列化和反序列化。

DataOutput和DataInput有一系列的write()和read()方法，注意写和读的顺序一定要一样。通过writeUTF(String)和readUTF()可以读写字符串。

下面是这个自定义类的代码：

```java
package cn.lazycat.bdd.hadoop.mapreduce.flow.update;

import org.apache.hadoop.io.Writable;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class FlowBean implements Writable {

    private String phone;
    private String addr;
    private String name;
    private long flow;

    @Override
    public void write(DataOutput out) throws IOException {

        out.writeUTF(phone);
        out.writeUTF(addr);
        out.writeUTF(name);
        out.writeLong(flow);
    }

    @Override
    public void readFields(DataInput in) throws IOException {

        phone = in.readUTF();
        addr = in.readUTF();
        name = in.readUTF();
        flow = in.readLong();
    }

    // getter, setter 略
}
```

随后修改map方法，此时输出的value就可以改为整个FlowBean对象了：

```java
public class UpdateFlowMapper
        extends Mapper<LongWritable, Text, Text, FlowBean> {

    @Override
    protected void map(LongWritable key, Text value,
                       Context context)
            throws IOException, InterruptedException {

        String line = value.toString();
        String temp[] = line.split(" ");

        String phone = temp[0];
        String addr = temp[1];
        String name = temp[2];
        long flow = Long.parseLong(temp[3]);

        FlowBean bean = new FlowBean();
        bean.setPhone(phone);
        bean.setAddr(addr);
        bean.setName(name);
        bean.setFlow(flow);

        context.write(new Text(phone), bean);
    }
}
```

reduce任务就接收phone和对象，然后把处理后的对象输出：

```java
public class UpdateFlowReduce
        extends Reducer<Text, FlowBean, Text, LongWritable> {
    @Override
    protected void reduce(Text key, Iterable<FlowBean> values,
                          Context context)
            throws IOException, InterruptedException {

        String info = key.toString();
        long sum = 0;
        for (FlowBean flowBean : values) {
            sum += flowBean.getFlow();
        }

        context.write(new Text(info), new LongWritable(sum));
    }
}
```

最终的结果和不使用自定义对象是一模一样的。

## Partitioner 分区

之前我们的全部MapReduce任务生成的都是一个文件。但是有时候我们可能想根据某些字段进行一些区分。

例如，上面的统计流量的案例。假如我想根据地区来形成不同的结果文件，例如，bj和sh流量使用情况放在不同的文件里面，就需要使用分区了。

分区操作是shuffle操作的一个重要过程，作用是把map的结果按照规则分发给不同的reduce进行处理，从而根据分区得到多个输出文件。

Partitioner是partitioner的基类，如果需要定制自己的分区类则可以继承这个类。

默认使用的分区是HashPartitioner。计算方法是：

```java
reducer = (key.hashCode() & Integer.MAX_VALUE) % numberReduceTasks;
```

这是一种随机分区，会把map任务产生的结果随机按照key的hash值随机分配给不同的reduce。

在默认情况下，reduceTask的数量是1。也就是说只有一个reduce，所以上面所有的MapReduce作业产生的最终结果只有一个。

### 分区案例：根据地区统计流量

下面改造统计流量的案例，按照不同的地区存放最终结果数据。

这需要我们自定义分区规则，按照addr进行分区，下面定义一个分区类：

```java
public class FlowPartitioner extends Partitioner<Text, FlowBean> {

    // 保存 城市-reducer编号
    private static Map<String, Integer> map;

    static {
        map = new HashMap<>();
        map.put("bj", 0);
        map.put("sh", 1);
        map.put("sz", 2);
    }

    @Override
    public int getPartition(Text text, FlowBean flowBean,
                            int numPartitions) {

        if (flowBean != null) {
            // 按照地址来分区
            String addr = flowBean.getAddr();
            return map.get(addr);
        }
        else {
            return 3;
        }

    }
}
```

getPartition会返回这个key-value会交给哪个reduce处理。返回的是reducer的编号(从0开始)。

在启动Job前，需要手动指定这个分区类：

```java
// 设置自定义分区类
job.setPartitionerClass(FlowPartitioner.class);
// 设置分区数量
job.setNumReduceTasks(4);
```

注意在设置分区数量的时候，一定要确保它是大于或等于你的分区类能够返回的最大编号+1的，否则会抛出异常(例如你返回了一个4而你的分区数量也是4)

MapReduce代码不需要改变，下面执行作业，再查看作业输出的结果：

```text
[root@hadoop1 ~]# hdls /out
Found 5 items
-rw-r--r--   3 LazyCat supergroup          0 2018-03-24 23:02 /out/_SUCCESS
-rw-r--r--   3 LazyCat supergroup         20 2018-03-24 23:02 /out/part-r-00000
-rw-r--r--   3 LazyCat supergroup         41 2018-03-24 23:02 /out/part-r-00001
-rw-r--r--   3 LazyCat supergroup         21 2018-03-24 23:02 /out/part-r-00002
-rw-r--r--   3 LazyCat supergroup          0 2018-03-24 23:02 /out/part-r-00003
```

会看到有4个输出结果。那么，0就表示"bj"城市的结果；1表示"sh"城市的结果；2表示"sz"城市的结果；4表示flowBean为空时的结果(错误的结果)。

## sort 排序

在shuffle的过程中，除了分区，我们还可以进行排序。

在默认情况下，是按照key的字典顺序进行排序的。我们可以自定义这个规则。

在Map执行过后，数据进入reduce之前，数据会被进行排序。

### 排序案例：计算每个人的总收益

现在假设我们有一个计算每个人总收益(收入 - 成本)的场景，并且，需要按照收益进行排序。

数据的形式如下：

- profit

```text
1 ls 2850 100
2 ls 3566 200
3 ls 4555 323
1 zs 19000 2000
2 zs 28599 3900
3 zs 34567 5000
1 ww 355 10
2 ww 555 222
3 ww 667 192
```

其中，第一列表示月份，第二列表示姓名，第三列表示收入，第四列表示成本。

要想实现这个功能，需要用到两个MapReduce作业。第一个用于统计每个人的总收入，第二个用于排序。

第一个任务非常简单，map以名字作为key，单月收益作为value输出，reduce把所有名字的单月收益汇总输出。

以下是这个任务map和reduce的代码片段：

```java
// map:
@Override
protected void map(LongWritable key, Text value,
                    Context context)
        throws IOException, InterruptedException {

    String data[] = value.toString().split(" ");
    String name = data[1];
    long profit = Long.parseLong(data[2]) - Long.parseLong(data[3]);

    context.write(new Text(name), new LongWritable(profit));
}
// reduce:
@Override
protected void reduce(Text key, Iterable<LongWritable> values,
                        Context context)
        throws IOException, InterruptedException {

    long sum = 0;
    for (LongWritable val : values) {
        sum += val.get();
    }

    context.write(key, new LongWritable(sum));
}
```

第一次MapReduce的计算结果：

```text
[root@hadoop1 ~]# hdcat /out/part-r-00000
ls  10348
ww  1153
zs  71266
```

随后我们需要编写第二次MapReduce进行排序。

我们可以写一个ProfitBean类，让这个类实现WritableComparable接口。这个接口同时继承了Writable和Comparable接口。然后在Shuffle的时候，如果把这个对象作为key，就会按照用户自定义的规则进行排序。

```java
package cn.lazycat.bdd.hadoop.mapreduce.profit;

import org.apache.hadoop.io.WritableComparable;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class ProfitBean
    implements WritableComparable<ProfitBean> {

    private String name;
    private long profit;

    @Override
    public int compareTo(ProfitBean o) {
        return (int) (o.getProfit() - profit);
    }

    @Override
    public void write(DataOutput out) throws IOException {

        out.writeUTF(name);
        out.writeLong(profit);

    }

    @Override
    public void readFields(DataInput in) throws IOException {

        name = in.readUTF();
        profit = in.readLong();
    }

    @Override
    public String toString() {
        return name + "\t" + profit;
    }

    // setter, getter 略

}
```

那么这个MapReduce就只负责一个shuffle阶段的排序了。在map作业我们需要把每一行转换为ProfitBean对象，在reduce阶段只是需要简单地输出这个对象即可：

```java
// map
@Override
protected void map(LongWritable key, Text value,
                    Context context)
        throws IOException, InterruptedException {
    String data[] = value.toString().split(" ");
    ProfitBean bean = new ProfitBean();
    bean.setName(data[0]);
    bean.setProfit(Long.parseLong(data[1]));
    context.write(bean, NullWritable.get());
}
// reduce
@Override
protected void reduce(ProfitBean key, Iterable<NullWritable> values,
                        Context context)
        throws IOException, InterruptedException {

    context.write(new Text(key.toString()), NullWritable.get());

}
```

结果就是已经排序好的了：

```text
[root@hadoop1 ~]# hdcat /out/part-r-00000
zs  71266
ls  10348
ww  1153
```

由此可见有时候一个MepReduce并不能满足我们的需求，我们可能会编写很多的MapReduce作业。

上面这个MapReduce作业我们甚至可以删去Reduce作业，因为Reduce作业做的事情实在是太少了，可以直接把Map的结果输出。

## Combiner 合并

因为Map作业是对每一行操作的。所有每一个Map任务都可能产生大量的输出。Combiner的作用是在Map作业时先进行合并，以减少到Reduce作业的工作量。

Combiner的特点就是在Map作业进行后进行一些"迷你的"合并(注意不是将多个key合并为一个，而是把同一个key的多个values进行合并)。但是它不能对所有数据进行合并。

### 升级WordCount：使用Combiner

之前的WordCount，在Map端输出的单词字数必定是1。这样Reduce需要对大量的个数为1的单词进行归并，显得有点鸡肋。如果我们能在Map端对一些数据进行一些归并，会大大减少Reduce端的压力。

combiner的写法和Reducer是几乎一样的，它们都需要继承Reducer类，甚至逻辑都是一模一样的：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount.combiner;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class WordCountCombiner
        extends Reducer<Text, LongWritable, Text, LongWritable> {


    @Override
    protected void reduce(Text key, Iterable<LongWritable> values,
                          Context context)
            throws IOException, InterruptedException {

        long sum = 0;
        for (LongWritable val : values) {
            sum += val.get();
        }

        context.write(key, new LongWritable(sum));
    }
}
```

接下来我们需要在启动类中申明使用这个Combiner：

```java
// 指定Combiner
job.setCombinerClass(WordCountCombiner.class);
```

运行作业，会发现结果是一模一样的。

## Shuffle 洗牌

Shuffle是MapReduce的"心脏"。它的作用是对Map作业的输出进行key归并、key排序，然后送给Reduce作业作为输入。但是Shuffle还有一些更为复杂的机制。

每个Map任务都有一个环形缓存区，默认情况下是100M(由io.sort.mb设置)，一旦缓存区内容达到阀值(io.sort.spill.percent)，默认为0.80，指的是最大大小的0.8倍。一个后台线程会被启动，把缓存区内容溢出(spill)到磁盘中。此时Map会继续向缓存区写数据。但是如果缓存区满了，Map作业会被阻塞。

这个过程非常有趣，可以把缓冲区想象为一个"圆盘"。这个圆盘上面有两个柱头。一个柱头滑过圆盘产生数据，另一个滑过圆盘移除数据。一开始，产生数据的柱头先行动，在快把整个圆盘写满数据时，移除数据的柱头开始移动，"追赶"写数据的柱头。如果写数据的柱头把整个圆盘写满数据了，就会停下来，等待读取数据的柱头把数据都读取了再动。

溢出写出的过程会轮询把数据写到mapred.local.dir的目录中。

在溢出前，线程会首先把溢出的数据按照分区规则进行分区，并且在分区中，会按照key进行"内排序"。

如果用户指定了Combiner，则会在"内排序"后对数据进行归并，使得Map作业的结果更加紧凑。

每次数据达到缓存区的阀值，都会产生一个新的"溢出文件"。因此在Map任务结束后可能会产生多个溢出文件。在Shuffle过程结束前，会把这些文件进行合并。合并后的文件数据是分区好，排序好的。

如果溢出文件超过3个，会在合并溢出文件之后再进行一次Combine。如果小于3，则Hadoop认为没有必要进行一次Combine了。

也就是说，Combiner在Shuffle中可能会被反复运行，以尽可能保证Map作业结果的紧凑。

Reducer通过HTTP获取溢出文件的分区，并获取这些分区的数据。

对于没有溢出的数据，则直接在内存交给Reduce作业处理。

## InputFormat

我们已经把整个Map、Shuffle、reduce过程介绍完了，下面的问题是，MapReduce是如何读取、写出数据文件的？

MapReduce使用InputFormat读取数据。

在MapReduce的开始阶段，InputFormat用于产生InputSplit(逻辑切块)，并把它切分成recorder，形成Mapper的输入。

org.apache.hadoop.InputFormat本身是一个接口，含有如下方法：

```java
/**
 * 对文件进行逻辑切块。
 * 注意这没有进行物理切块，InputSplit包括了文件的位置、从哪开始切，到哪结束等信息。
 */
InputSplit[] getSplits(JobConf job);

/**
 * 指明了读取的方式，也就是一次读取多少内容，以及如何读取这些数据。
 * RecorderReader返回这次读取了多少内容，以及读取的Key和Value等。也就是读取的具体内容
 */
RecordReader<K, V> getRecordReader(InputSplit split, JobConf job);
```

MapReduce默认使用InputFormat的TextInputFormat作为文件读取类。这个类又是FileInputFormat的子类。

除了TextInputFormat，还有KeyValueTextInputFormat。同样用于读取文件，如果行被分隔符（默认是tab），则按照分隔符把每行拆分成key和value。

SequenceFileInputFormat用于读取sequence file。这是Hadoop用于储存数据自定义格式的binary文件。它有两个子类：SequenceFileAsBinaryInputFormat，这将key和value以BytesWritable的类型输出。SequcenFileAsTextInputFormat将key和value以Text形式输出。

SequenceFileInputFilter根据filter从sequence文件中取得部分满足条件的数据，通过setFilterClass指定Filter。Filter有三种内置的：RegexFilter,PercentFilter,MD5Filter。Regex可以根据正则表达式进行过滤，Percent通过指定参数f取记录行数被f整除的行；MD5通过指定参数，取 MD5()

如果要自定义切分规则，并且读取的是文件，可以继承FileInputFormat。

FileInputFormat实现了geiSplits()方法。它会遍历Job下的文件，统计所有文件的大小。

### 自定义InputFormat案例：统计成绩

有时候，默认的InputFormat可能满足不了我们的需求，一个典型的例子是一次读取多行数据。

例如我们现在要统计学生的总分，数据格式如下：

```text
张三
语文 97
数学 77
英语 69
李四
语文 87
数学 57
英语 63
王五
语文 47
数学 54
英语 39
```

Map任务的时候一次需要读取4行数据。显然默认的InputFormat无法满足我们的需求。

而如果要我们手动实现一个逻辑切块，代码会过于复杂，所以我们一般间接继承。例如这个例子我们可以继承FileInputFormat，这是所以对文件操作的基类。这样逻辑切块的代码就不需要我们自己写了。

我们需要自己实现createRecordReader()方法。这个方法返回的RecordReader对象可以用来指明一次读取多少行数据，里面需要包含读取的key和value。

RecordReader是一个抽象类，有非常多方法需要我们自己实现，下面是实现4行读的实现类代码：

```java
package cn.lazycat.bdd.hadoop.mapreduce.format;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.RecordReader;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.input.FileSplit;
import org.apache.hadoop.util.LineReader;

import java.io.IOException;

public class ScoreRecordReader extends RecordReader<Text, Text> {

    // 读取文件的读行器
    private LineReader reader;

    // 每行的key
    private Text key;

    // 每行的value
    private Text value;

    // 因为一次性要读取多行，所以需要一个分隔符来区分不同行的数据
    private static String separator = "|";

    // 是否含有下一个数据
    private boolean hasNext = true;

    /**
     * 初始化RecordReader
     */
    @Override
    public void initialize(InputSplit split, TaskAttemptContext context)
            throws IOException, InterruptedException {

        // 取得切块对象
        FileSplit fileSplit = (FileSplit) split;
        // 取得切块
        Path path = fileSplit.getPath();

        // 配置对象
        Configuration conf = new Configuration();
        // 取得文件系统对象
        FileSystem fs = path.getFileSystem(conf);
        // 在文件系统中取得要操作的文件流
        FSDataInputStream in = fs.open(path);
        // 创建读行器对象
        // BufferedReader reader = new BufferedReader(new InputStreamReader(in));
        reader = new LineReader(in);

    }

    /**
     * 判断是否存在下一个key-value。
     * 如果有，为key和value赋值。
     */
    @Override
    public boolean nextKeyValue() throws IOException {
        // 需要实例化一对key-value
        key = new Text();
        value = new Text();

        // 一次性读取4行
        Text cur = new Text();
        int len;
        for (int i = 0; i < 4; ++i) {
            len = reader.readLine(cur);
            if (len == 0) {  // 表示读取到最后一行了
                hasNext = false;   // 已经没有数据了
                break;
            }
            else {
                if (i == 0) {  // 4行中的第一行，应当是key值
                    key.set(cur);
                }
                else {        // 4行中的其余行，应该追加到value中
                    // 追加到尾端，以separator作为分割
                    value.append(cur.getBytes(), 0, cur.getLength());
                    value.append(separator.getBytes(), 0,
                            separator.length());
                }
            }
            cur.clear();
        }
        return hasNext;
    }

    /**
     * 返回当前的key
     */
    @Override
    public Text getCurrentKey() {
        return key;
    }

    /**
     * 返回当前的value
     */
    @Override
    public Text getCurrentValue() {
        return value;
    }

    /**
     * 返回当前的进度
     */
    @Override
    public float getProgress() {
        return hasNext ? 0f : 1f;
    }

    /**
     * 关闭执行的方法
     */
    @Override
    public void close() throws IOException {
        reader.close();
    }
}
```

自定义InputFormat代码：

```java
package cn.lazycat.bdd.hadoop.mapreduce.format;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.InputSplit;
import org.apache.hadoop.mapreduce.RecordReader;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;

public class ScoreInputFormat extends FileInputFormat<Text, Text> {
    @Override
    public RecordReader<Text, Text> createRecordReader(InputSplit split,
               TaskAttemptContext context) {
        return new ScoreRecordReader();
    }
}
```

这样一来，Map任务的输入就是Text-Text对了，key是学生的姓名，value是学生的所有成绩。

下面的map的代码(不需要reduce)：

```java
@Override
protected void map(Text key, Text value, Context context)
        throws IOException, InterruptedException {

    String name = key.toString();
    double sum = 0;
    String data[] = value.toString().split("\\|");

    for (String str : data) {
        String scoreStr = str.split(" ")[1];
        sum += Double.parseDouble(scoreStr);
    }
    context.write(new Text(name), new DoubleWritable(sum));
}
```

在启动类之中需要注册这个InputFormat：

```java
job.setInputFormatClass(ScoreInputFormat.class);
```

随后，就可以看到输出：

```text
[root@hadoop1 ~]# hdcat /out/part-r-00000
张三    243.0
李四    207.0
王五    140.0
```

## MultipleInputs

MultipleInputs可以将多个输入组装起来，并向Map提供数据。

此类提供有静态方法：

```java
// 指定数据来源以及InputFormat
addInputPath(job, path, inputFormatClass);
// 指定数据来源，InputFormat和Mapper
addInputPath(job, path, inputFormatClass, mapperClass);
```

如果我们的输入在不同的目录，可以通过调用这个方法来多方导入。我们可以使用不同的InputFormat来处理不同源的数据。

我们甚至可以通过指定不同的mapperClass来让不同的mapper来处理不同目录下的文件。

## OutputFormat

在MapReduce结束前，OutputFormat决定了Reduce作业如何产生输出。

Hadoop本身提供了很多默认的OutputFormat，如果不指明的话默认使用TextOutputFormat。

常见的OutputFormat：

- FileOutputFormat: 实现了OutputFormat接口，用于输出文件。
  - MapFileOutputFormat: 使用部分索引的形式输出。
  - SequenceFileOutputFormat: 二进制键值数据压缩形式输出。
  - SequenceFileAsBinaryOutputFormat: 原生二进制压缩格式。
  - TextOutputFormat: 以行作为分割，包含制表符定界的键值对文本文件格式。
  - MultipleOutputFormat: 使用键值对参数写入文件的抽象类。
    - MultipleTextOutputFormat: 输出多个以标准行分割，制表符定界格式的文件。
    - MultipleSequenceFileOutputFormat: 输出多个压缩文件。

可以通过job.setOutputFormatClass(outputFormatClass)来指定使用的OutputFormat。

有时候默认的OutputFormat无法满足我们的需求，需要我们自定义OutputFormat。和InputFormat一样，我们一般继承FileOutputFormat这个抽象类。

OutputFormat默认包含了getOutputCommitter()和checkOutputSecs()方法，这两个方法FileOutputFormat已经棒我们实现好了。如果想要更加精确的逻辑控制，可以覆写。

然而一般情况下我们只对文件输出的格式进行改写。所以只需要覆写getRecordWriter()方法。

RecordWriter对象和RecordReader对象很像，是用来规定输出的格式的。

### WordCount升级

下面对WordCount任务进行一个升级，使得输出的格式是key1-value1#key2-value2#...。输出的格式为一行。

国际惯例，先定义RecordWriter类：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount.format;

import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;

import java.io.IOException;

public class WordCountRecordWriter extends RecordWriter<Text, LongWritable> {

    private FSDataOutputStream out;

    /**
     * 需要接收来自FS的输出流
     */
    public WordCountRecordWriter(FSDataOutputStream out) {
        this.out = out;
    }

    @Override
    public void write(Text key, LongWritable value) throws IOException {
        // 根据业务需求输出
        out.writeUTF(key.toString() + "-" + value + "#");
    }

    @Override
    public void close(TaskAttemptContext context) throws IOException {
        out.close();
    }
}
```

随后定义OutputFormat类：

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount.format;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.RecordWriter;
import org.apache.hadoop.mapreduce.TaskAttemptContext;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class WordCountOutputFormat extends FileOutputFormat<Text, LongWritable> {

    @Override
    public RecordWriter<Text, LongWritable> getRecordWriter(TaskAttemptContext job)
            throws IOException {
        // 获取conf对象
        Configuration conf = job.getConfiguration();
        // 获取path
        Path path = getDefaultWorkFile(job, "");
        // 获取文件系统对象
        FileSystem fs = path.getFileSystem(conf);

        FSDataOutputStream out = fs.create(path, false);

        return new WordCountRecordWriter(out);
    }
}
```

当然，需要在启动类中注册这个OutputFormat：

```java
job.setOutputFormatClass(WordCountOutputFormat.class);
```

随后执行，输出的格式如下：

```text
I-3#a-1#am-1#cat-3#hadoop-2#hello-3#love-2#small-1#world-1#
```

## MultipleOutputs

MultipleOutputs可以使一个Reduce作业产生多个输出。

我们需要使用这个类的一个普通方法和一个静态方法：

```java
// 增加一个输出，需要指定标记和outputFormat
static addNamedOutput(job, "flag", outputFormat);
// 设定输出文件，flag表示输出文件的标记，key和value表示输出的内容
write("flag", key, value);
```

要想使用这个类，需要在Reducer中保存这个对象的实例。随后需要覆写Reducer的setup()方法，初始化这个实例：

```java
@Override
protected void setup(context) {
    super(context);
    outputs = new MultipleOutputs();
}
```

可以根据业务需要在reduce方法中产生不同的输出文件。这时候不再调用context.write()输出了，而是调用outputs.write()输出

```java
outputs.write("flag", key, value);
```

在启动类需要这个MultipleOutputs:

```java
MultipleOutputs.addNamedOutput(job, "flag", outputFormat);
```

注意这里的标记需要和Reducer的标记一一对应。

这样，我们可以实现对不同的输出文件进行不同的OutputFormat。

这样输出文件的文件名称就类似于：flag-r-00000这样了。

## GroupingComparator 分组

默认情况下，在Map完成之后，会对key相同的value进行merge(key1-value1, key1-value2 -> key1-(value1,value2))。这个过程也叫Grouping。

我们也可以自己指定Grouping的规则。只需要编写一个继承GroupingComparator的类即可。

我们可以使用GroupingComparator的子类WritableComparator类。实现其中的compare方法。

这个方法的原型如下：

```java
public int compare(byte[] b1, int s1, int l2, byte[] b2, int s2, int l2);
```

b1和b2表示Map作业输出的两个key（注意是序列化后的数据）。我们需要指定key比较的规则。

### 案例：统计a-n，o-z单词个数的统计

本案例就是在WordCount的基础上修改分组的规则。这需要我们自己定义一个分组类。

```java
package cn.lazycat.bdd.hadoop.mapreduce.wordcount.grouping;

import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.WritableComparator;

import java.io.ByteArrayInputStream;
import java.io.DataInput;
import java.io.DataInputStream;
import java.io.IOException;

public class WCGroupingComparator extends WritableComparator {

    @Override
    public int compare(byte[] b1, int s1, int l1, byte[] b2, int s2, int l2) {
        Text key1 = new Text();
        Text key2 = new Text();

        // 对b1和b2进行反序列化
        deSerial(key1, b1, s1, l1);
        deSerial(key2, b2, s2, l2);

        // 如果key1和key2都是以a-n或者o-z开头的，认为它们相等
        if (chargeRegEquals(key1, key2, "[a-n][a-z]*")
                || chargeRegEquals(key1, key2, "[o-z][a-z]*")) {
            return 0;
        }
        else {  // 其它情况都认为不相等
            return -1;
        }
    }

    // 对字节数组进行反序列化
    private void deSerial(Text text, byte[] b, int s, int l) {
        DataInput in;
        try {
            in = new DataInputStream(new ByteArrayInputStream(b, s, l));
            // 反序列化
            text.readFields(in);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // 判断两个text是否都符合某个正则表达式
    private boolean chargeRegEquals(Text text1, Text text2, String regex) {
        return text1.toString().matches(regex)
                && text2.toString().matches(regex);
    }
}
```

在启动类中注册这个分组：

```java
job.setGroupingComparatorClass(WCGroupingComparator.class);
```

## SortComparator Reduce排序

一般来说，在Map任务进行的时候就会进行排序操作了。所以我们很少会在Reduce任务再进行排序。

但是还是可以这么做的。

这需要我们写一个WritableComparator实现类。这个类的写法和上面分组介绍的类似，区别是一定要区分出大小(compare返回1或-1)。

调用job的setSortComparator()可以注册这个Comparator。
