---
layout:       post
title:        "HBase 进阶笔记"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - HBase
---

## 高级查询

HBase的Java API提供了一些高级的查询功能。所谓的“高级”，其实一点也不高级，无非就是对HBase的表进行一些范围化的查询和数据的过滤，而不是用get仅取出一个行键的内容。

为了测试方便，我这里插入一些简单的测试数据，待会就是对这些数据进行查询：

```text
put 'tab1','rk1','cf1:c1','val1'
put 'tab1','rk1','cf1:c2','val2'
put 'tab1','rk2','cf1:c1','val3'
put 'tab1','rk3','cf1:c2','val4'
put 'tab1','rk4','cf1:c3','val5'
put 'tab1','rk5','cf1:c4','val6'
put 'tab1','rk6','cf1:c1','val7'
```

执行后tab1表变成了下面这样子：

rk | cf1:c1 | cf1:c2 | cf1:c3 | cf1:c4
---|---|---|---|---
rk1 | val1 | val2 | |
rk2 | val3 | | |
rk3 | | val4 | |
rk4 | | | val5 |
rk5 | | | | val6
rk6 | val7 | | |

很明显，这是一张比较稀疏的表。

### Scan 查询

首先，我们可以利用Scan进行全表扫描：

```java
package cn.lazycat.bdd.hbase.ext;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.HTable;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.client.ResultScanner;
import org.apache.hadoop.hbase.client.Scan;

import java.io.IOException;
import java.util.Map;
import java.util.NavigableMap;

public class HBaseAdvDemo1 {

    public static void main(String[] args) throws IOException {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "hadoop1:2181," +
                "hadoop2:2181,hadoop3:2181");
        HTable tab = new HTable(conf, "tab1".getBytes());

        // 创建scan对象
        Scan scan = new Scan();

        // 得到结果集
        ResultScanner rs = tab.getScanner(scan);

        // 遍历
        for (Result res : rs) {
            // 取得行键
            String rk = new String(res.getRow());

            // 取得每一列的数据
            // 注意三层泛型：
            // 1. byte[]表示列族的名称，Map表示列中储存的列信息
            // 2. byte[]表示列的名称，Map表示不同版本的列数据
            // 3. Long表示一个版本的时间戳，byte[]表示储存的具体数据
            NavigableMap<byte[], NavigableMap<byte[],
                    NavigableMap<Long, byte[]>>> map = res.getMap();

            // 遍历Map
            for (Map.Entry<byte[], NavigableMap<byte[],
                    NavigableMap<Long, byte[]>>> entry : map.entrySet()) {

                // 列族名称
                String cf = new String(entry.getKey());

                // 这个列族的多个列
                NavigableMap<byte[], NavigableMap<Long, byte[]>>
                        cmap = entry.getValue();

                // 遍历这个列族的所有列
                for (Map.Entry<byte[], NavigableMap<Long, byte[]>>
                        centry: cmap.entrySet()) {

                    // 列名称
                    String column = new String(centry.getKey());

                    // 这个列的多个版本
                    NavigableMap<Long, byte[]> dataMap =
                            centry.getValue();

                    // 遍历所有的数据版本
                    for (Map.Entry<Long, byte[]> dataEntry :
                            dataMap.entrySet()) {

                        long timestamp = dataEntry.getKey();
                        String data = new String(dataEntry.getValue());

                        // 打印，行键，列族，列，时间戳，具体数据
                        System.out.println("====== cell ======");
                        System.out.println("row key = " + rk);
                        System.out.println("column family = " + cf);
                        System.out.println("column = " + column);
                        System.out.println("timestamp = " + timestamp);
                        System.out.println("data = " + data);

                    }  // cell

                } // column

            }  // column family

        }  // row key

        tab.close();

    }

}
```

Result对象取出的Map看上去很复杂，但是如果你熟悉HBase的储存一张表的结构的话，就一点也不复杂了。

下面是这段代码的输出结果：

```text
====== cell ======
row key = rk1
column family = cf1
column = c1
timestamp = 1523600500719
data = val1
====== cell ======
row key = rk1
column family = cf1
column = c2
timestamp = 1523600524370
data = val2
====== cell ======
row key = rk2
column family = cf1
column = c1
timestamp = 1523600535396
data = val3
====== cell ======
row key = rk3
column family = cf1
column = c2
timestamp = 1523600547248
data = val4
====== cell ======
row key = rk4
column family = cf1
column = c3
timestamp = 1523600561272
data = val5
====== cell ======
row key = rk5
column family = cf1
column = c4
timestamp = 1523600589137
data = val6
====== cell ======
row key = rk6
column family = cf1
column = c1
timestamp = 1523600644097
data = val7
```

我们看到客户端正确地取得了表的所有数据。

我们也可以按照行键进行一些过滤。例如根据行键的范围进行查询，这只需要对scan对象调用一些方法即可，例如，加上下面两行代码就可以仅取出"rk2"-"rk4"行键的数据：

```java
scan.setStartRow("rk2".getBytes());
scan.setStopRow("rk5".getBytes());
```

因为行键是默认按照字典顺序排好的，所以我这里只需要指定开头和结尾即可实现范围查询。

注意是含头不含尾的，所以我这里的stop设置的是5。

加上这两行的执行结果：

```text
====== cell ======
row key = rk2
column family = cf1
column = c1
timestamp = 1523600535396
data = val3
====== cell ======
row key = rk3
column family = cf1
column = c2
timestamp = 1523600547248
data = val4
====== cell ======
row key = rk4
column family = cf1
column = c3
timestamp = 1523600561272
data = val5
```

### 过滤器

我们可以编写一个过滤器，对Scan查询的结果进行一些过滤的操作。

Scan对象有一个setFilter()方法，可以传入一个过滤器。这是一个抽象类。我们可以自定义Filter或者使用官方提供的Filter。

官方提供有以下常见的Filter：

RowFilter, 根据行进行过滤，需要传入比较类型和比较器。比较类型指定了按照什么规则进行比较，提供有等于、不等于等功能，比较器需要传入具体要比较的值。

例如，过滤行键为"rw1"的数据：

```java
// 创建过滤器
Filter filter = new RowFilter(CompareFilter.CompareOp.EQUAL,
        new BinaryComparator("rk1".getBytes()));
// 指定过滤器
scan.setFilter(filter);
```

执行结果：

```text
====== cell ======
row key = rk1
column family = cf1
column = c1
timestamp = 1523600500719
data = val1
====== cell ======
row key = rk1
column family = cf1
column = c2
timestamp = 1523600524370
data = val2
```

CompareOp除了EQUAL还有NOT_EQUAL、GREATER、GREATER_OR_EQUAL、LESS、LESS_OR_EQUAL，能够按照大小进行比较。

除了BinaryComparator，我们可以使用RegexStringComparator。注意使用这个Comparator则ComparaOp只能使用EQUAL或NOT_EQUAL。顾名思义，这个Comparator可以让行键按照正则表达式进行过滤。

我们可以增加两个行键作为示例：

```text
put 'tab1','hhh1','cf1:c1','nihao!'
put 'tab1','hhh2','cf1:c1','nihao!'
```

假如我想过滤出行键包含"hhh"的单元格，可以加上下面的两行代码：

```java
Filter filter = new RowFilter(CompareFilter.CompareOp.EQUAL,
        new RegexStringComparator("^.*hhh.*$"));
scan.setFilter(filter);
```

输出结果：

```text
====== cell ======
row key = hhh1
column family = cf1
column = c1
timestamp = 1523604567845
data = nihao!
====== cell ======
row key = hhh2
column family = cf1
column = c1
timestamp = 1523604583926
data = nihao!
```

ValueFilter可以按照数据值筛选数据，用法和RowFilter一样。

除了RowFilter，还有PrefixFilter，可以筛选出有特定前缀的行键的数据。用法比较简单，下面查询出有"hhh"前缀的数据：

```java
Filter filter = new PrefixFilter("hhh".getBytes());
scan.setFilter(filter);
```

执行结果和上一次是一样的。

ColumnPrefixFilter可以根据列前缀(注意需要包含列族)进行过滤。用法和PrefixFilter一样，不再演示。

KeyOnlyFilter可以只返回数据的行键，列，时间戳，不返回具体的数据。如果只是想了解一张表的结构而不想获取里面的数据，可以使用这个过滤器，以节约网络带宽。

下面是一个示例：

```java
scan.setStartRow("rk1".getBytes());
scan.setStopRow("rk3".getBytes());

Filter filter = new KeyOnlyFilter();
scan.setFilter(filter);
```

输出结果：

```text
====== cell ======
row key = rk1
column family = cf1
column = c1
timestamp = 1523600500719
data =
====== cell ======
row key = rk1
column family = cf1
column = c2
timestamp = 1523600524370
data =
====== cell ======
row key = rk2
column family = cf1
column = c1
timestamp = 1523600535396
data =
```

RandomRowFilter可以从结果集中随机取出结果。对于同一个结果集，这个过滤器的执行结果可能是不一样的。如果想实现一些随机的抽样，可以使用这个过滤器：

```java
Filter filter = new RandomRowFilter(0.3f);
scan.setFilter(filter);
```

其中，需要传入一个float，表示每一条数据被返回出的概率。

因为每次的执行结果都不一样，所以这里不再展示运行结果。

InclusiveStopFilter可以设置返回到哪个行键。和Scan的setStopRow()方法不同，这个过滤器是包含结束行键的。

我们可以利用下面代码实现对"rk2"-"rk4"行键的查询：

```java
scan.setStartRow("rk2".getBytes());
Filter filter = new InclusiveStopFilter("rk4".getBytes());
scan.setFilter(filter);
```

执行结果和之前的范围查询是一样的。

FirstKeyOnlyFilter可以仅返回每一行的第一列数据：

```java
scan.setStartRow("rk1".getBytes());
scan.setStopRow("rk3".getBytes());

Filter filter = new FirstKeyOnlyFilter();
scan.setFilter(filter);
```

输出结果：

```text
====== cell ======
row key = rk1
column family = cf1
column = c1
timestamp = 1523600500719
data = val1
====== cell ======
row key = rk2
column family = cf1
column = c1
timestamp = 1523600535396
data = val3
```



SingleColumnFilter可以按照某一个列的值决定是否过滤这一行数据。注意setFilterIfMissing(boolean)方法可以设置是否过滤掉列不存在的行。默认为false，表示不过滤。也就是默认如果某行中这一列数据不存在，还是会被放到结果集里头。

例如我们过滤出"cf1:c1"下值为"val3"的行，如果在某行中这列数据不存在，不予显示：

```java
SingleColumnValueFilter filter = new SingleColumnValueFilter(
        "cf1".getBytes(), "c1".getBytes(),
        CompareFilter.CompareOp.EQUAL,
        new BinaryComparator("val3".getBytes())
);
filter.setFilterIfMissing(true);
scan.setFilter(filter);
```

输出结果：

```text
====== cell ======
row key = rk2
column family = cf1
column = c1
timestamp = 1523600535396
data = val3
```

还有很多其它的过滤器，这里不再演示。

如果我们需要增加多个过滤器，可以使用FilterList。用法如下：

```java
FilterList filters = new FilterList();
filters.addFilter(filter1);
filters.addFilter(filter2);
filters.addFilter(filter3);
...
scan.setFilter(filters);
```

FilterList可以接收一个Operator(构造)，有两个选择：

- MUST_PASS_ALL: 必须满足所有过滤器才显示结果
- MUST_PASS_ONE: 只需要满足至少一个过滤器即可

默认使用的是MUST_PASS_ALL。

## 表设计

如何设计HBase的表会直接影响HBase的使用效率和便利性。主要讨论的是列族和行键的设计。

对于列族，我们不应该设置太多，是越少越好的，官方建议列族数量不宜超过3个。在查询的时候，应该减少跨列族的访问。一般设置为1个，最多不要超过2个。如果有多个列族，多个列族的数据应该尽量设计得均匀。

对于行键，需要遵循以下原则：

- 行键必须唯一，硬性规定。
- 行键最好有意义，最好不要用随机值作为行键，一般使用常用作查询的字段作为行键。
- 行键最好是字符串，最好不要使用数值，因为不同系统处理数值的方式可能不同。
- 行键最好定长，因为字典排序的原因，如果不定长，排序结果可能和预期不一致。
- 行键不宜过长，HBase最大支持64KB行键，但是最好不要超过100bytes。并且最好是8的整数倍。

行键不宜过长是因为最终储存在storeFile中的数据是键值对，行键会被不停地重复保存，如果行键过长，那么会占用大量的空间去储存行键。

通过以下的原则可以确定行键的顺序：

- 散列：为了防止某个Region成为热点而导致大多数查询都集中在一个RegionServer上，导致集群的性能分配不均匀。所以热点数据最好分开来存放。
- 有序：在设计行键的时候，应该将经常需要连续查询的数据的行键按照顺序排好。以加快热点数据的查询。

散列和有序在一定程度上有矛盾。在实际中我们应该根据数据特点进行取舍。

在实际中，一般把常需要查询的热点字段拼接在行键中。我们可以在行键中增加随机值以达到行键定长的效果（如果其它字段是不定长的，可以动态改变随机值的长度），如果要遵循散列原则，可以把随机值放在前面，如果遵循有序原则，可以把热点的字段放在前面。

## Phoenix

HBase作为一款数据库，无法用SQL去操作是一大遗憾。于是就诞生了Phoenix。这是一款中间件工具，可以使用类似SQL的语法和JDBC去操作HBase。

安装Phoenix步骤很简单，解压即可，注意在官网留意Phoenix和HBase的版本对照。

之后需要把phoenix-server和phoenix-Client拷贝到HBase的lib目录下。这两个文件在Phoenix的根目录下。

```text
[root@hadoop1 develop]# cd phoenix-4.8/
[root@hadoop1 phoenix-4.8]# cp phoenix-4.8.1-HBase-0.98-server.jar /usr/develop/hbase0.98/lib/
[root@hadoop1 phoenix-4.8]# cp phoenix-4.8.1-HBase-0.98-client.jar /usr/develop/hbase0.98/lib/
```

之后需要增加HBASE_HOME环境变量，指定HBase的根目录。

在HBase启动后，执行bin下的sqlline.py即可。需要传入你HBase集群节点的ZK信息：

```text
[root@hadoop1 develop]# cd phoenix-4.8/
[root@hadoop1 phoenix-4.8]# bin/sqlline.py hadoop1,hadoop2,hadoop3:2181
...
0: jdbc:phoenix:hadoop1,hadoop2,hadoop3:2181>
```

如果启动的过程中报错，一般重启HBase即可。

Phoenix会在HBase下新建一个SYSTEM名称空间，建立一些表，一般我们不去动它。

建表的操作如下：

- create table tab(field type primary key[, field type]...);

注意必须有一个主键，这个主键作为HBase的行键存在。

下面新建这么一张表：

```text
create table tab2(id integer primary key, name varchar);
```

在HBase中可以看到创建的表：

```text
hbase(main):001:0> list
...

=> ["SYSTEM.CATALOG", "SYSTEM.FUNCTION", "SYSTEM.SEQUENCE", "SYSTEM.STATS", "TAB2", "tab1"]
```

在Phoenix可以使用"!describe tab"来查看表的结构。

phoenix有以下特点：

- 在Phoenix下创建表会在HBase也创建表。
- 在Phoenix中创建的表列名称会变成大写，如果不想变大写可以使用双引号括起来。
- Phoenix中创建的主键会变成HBase的行键。默认情况下列会存放到HBase的列族"0"下。我们可以在Phoenix创建表的时候在列名前加"cf."来显示声明储存在哪个列族中。

如果要操作HBase已经事先存在的表，需要在Phoenix中手动创建，注意创建的时候列需要指明列族："cf"."c"。所有来自HBase的列和列族都需要用双引号括起来。

Phenix插入数据的语法如下：

- upsert into tab values(val1,val2,...);

这个命令既可以实现增加数据，也可以实现修改数据。val的值如果是字符串，需要用单引号括起来。val可以为null，表示没有数据。

查询语句和sql类似，仅支持简单查询：

- select xxx from tab where xxx;

注意where中指定列名称需要用双引号括起来。例如"name"='zhangsan'

删除的操作也是类似的：

- delete from tab where xxx;

注意这个操作会删除HBase底层的表。

我们可以创建视图，从而防止HBase底层表的删除。

- 创建视图：create view view_name as select xxx from tab;
- 查询视图：select xxx from view_name where xxx;
- 删除视图：drop view view_name;
