#### HBase与RDBMS

HBase是分布式的、面向列的数据存储系统，基于HDFS提供随机的读写服务。可横向扩展（百万列），也可以纵向扩展（数十亿行），并且可以跨数千个节点自动地水平地分区（区域）和备份。表的schema反映了物理存储，可以高效地进行数据结构序列化、存储和获取

一般的RDBMS是固定模式的（fixed-schema）、面向行的数据库，具有ACID属性和复杂的SQL查询引擎。侧重点在强一致性和参照完整性、从物理层抽象、用SQL执行复杂的查询。可以简单地创建索引、执行复杂的内外连接、对列聚合、对数据分页等等。

但是HBase的优势在于可扩展、并发读写。

HBase的一些特性：

* 没有真实的索引

  行存储是有顺序的，每行中的列也是有顺序的。因此没有索引膨胀的问题，插入性能与表的大小无关。

* 自动分区

  随着表的增长，会自动地分为区域，并把区域分发到所有可用的节点

* 线性扩展、并且自动启用新的节点

  增加一个节点，将它执行既存的集群，运行regionserver。区域会自动地再平衡，负载会均匀分布

* 普通商业硬件

  RDBMS硬件的IO要求很高，硬件成本昂贵。HBase集群节点只用普通硬件即可。

* 容错

  多节点意味着每个节点的重要性相对较低。

* 批处理

  与MapReduce集成，可以充分地并行、分布式的运行jobs。

#### HDFS

MapReduce使用HDFS的时候，通过map task打开HDFS文件流式读取HDFS文件内容，然后关闭HDFS文件。而在HBase中，随着集群的启动HDFS中的数据文件就被打开，并且一直保持打开以避免每次访问数据时打开文件的相关开销。因此，HBase可能会遇到一般MapReduce客户端不会遇到的问题：

* descriptors耗尽

  因为保持文件开启，对于压力大的集群很快就会遇到系统或者Hadoop强制的限制。比如，集群有三个节点，每个节点一个datanode和一个regionserver，正在往目前有100区域和10个列族的表中上传数据。如果平均每个列族有两个刷出文件，那么同时会打开2000（100\*10\*2）文件。每个打开的文件至少消耗一个descriptor，再加上外部scanners和Java库消耗的descriptors……每个进程默认的descriptors数量是1024。当超过文件系统_ulimit_时，日志中会有“Too many open files”的信息，但是首先会发现HBase出现了不确定行为。通常设置_ulimit_为10240。

* datanode线程耗尽

  Hadoop datanode有一个同时可以运行的线程的上限。Hadoop 1默认值是256（_dfs.datanode.max.xcievers_）。Hadoop 2将默认值增加到了4096（_hdfs-site.xml_中的_dfs.datanode.max.transfer.threads_）。

#### HBase存储机制

HBase的表对应的是HDFS中的一个目录，目录名为HBase表名（如果不使用命名空间，则在HBase默认的_default_目录下），表目录下是区域名命名的目录，区域目录下是每个列族的目录，每个列族目录中存储的是实际的数据文件，数据文件以**HFile**的形式存在。

```
/hbase/data/default/<tb_name>/<region>/<column family>/<hfile>
```

#### HBase数据的载入

**Java客户端**

**MapReduce**

**BulkLoad**

使用Jar包`hbase-it`

rowKey热点数据，rowKey散列