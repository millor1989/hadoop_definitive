## 1、Installing Hive

通常，Hive在工作站上运行并且将SQL转换为一系列在Hadoop集群上执行的jobs。Hive把数据组织为Hive表，Hive表提供了一种连接表结构与HDFS中数据的方式。元数据（Metadata）——例如，表结构（table schemas）——保存在一个叫作_metadata_的数据库中。

安装Hive很直接简便。需要与Hadoop版本一致的Hive。下载一个release，解压tarball到工作站的合适位置：

```
% tar xzf apache-hive-x.y.z-bin.tar.gz
```

把Hive加到Path会使它很容易启动：

```
% export HIVE_HOME=~/sw/apache-hive-x.y.z-bin
% export PATH=$PATH:$HIVE_HOME/bin
```

输入_hive_启动Hive shell：

```
% hive
hive>
```

### 1.1、The Hive Shell

Hive shell是与Hive交互的主要方式，通过发出_HiveQL_命令的方式。HiveQL是Hive的查询语言，一种SQL的方言。它深受MySQL影响，与MySQL类似。与SQL类似，HiveQL一般是大小写不敏感的，通过Tab键可以自动完成Hive关键字和函数，HiveQL命令必须以分号结尾。

第一次启动Hive时，可以通过查询表列表命令检查Hive是否在运行：

```
hive> SHOW TABLES;
OK
Time taken: 0.473 seconds
```

对于刚安装的Hive，命令会延时一些运行，因为它需要在机器上创建metastore数据库。（这个数据库把它的文件保存在一个叫做_metastore\_db_的目录中，这个目录位置与运行_hive_命令的位置有关。）

也可以用非交互式的方式运行Hive shell。**-f**选项运行指定文件中的命令：

```
% hive -f script.q
```

也可以用**-e**选项运行指定的命令，此时不需要最后的分号：

```
% hive -e 'SELECT * FROM dummy'
OK
X
Time taken: 1.22 seconds, Fetched: 1 row(s)
```

创建一个只有一行的表的简单例子：

```
% hive -e "CREATE TABLE dummy (value STRING); \
  LOAD DATA LOCAL INPATH '/tmp/dummy.txt' \
  OVERWRITE INTO TABLE dummy"
```

不管是交互式还是非交互式的Hive shell，Hive会把操作过程中的信息——比如运行查询花费的时间——打印到标准错误输出。使用**-S**选项可以禁止这些信息，结果是，只会输出查询的结果：

```
% hive -S -e 'SELECT * FROM dummy'
X
```

给命令加一个**!**前缀会在host操作系统上运行命令，比如在Hive shell交互中运行Hive所在操作系统（Linux）shell命令；使用**dfs**命令可以访问Hadoop文件系统。

## 2、An Example

创建气象数据表保存气象数据：

```sql
CREATE TABLE records (year STRING, temperature INT, quality INT)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t';
```

往表中加载数据：

```sql
LOAD DATA LOCAL INPATH 'input/ncdc/micro-tab/sample.txt'
OVERWRITE INTO TABLE records;
```

运行这个命令，Hive把指定的本地文件放进它的仓库目录。这是一个简单的文件系统操作。不会解析文件也不会

以内部数据库格式保存这个文件，因为，Hive不会限定任何特定的输入文件格式。文件会被逐字保存，并且不会被Hive修改。

这个例子中，把Hive表保存在本地文件系统（_fs.defaultFS_被设置为默认值**file:///**）。表被保存为Hive的仓库目录的子目录，仓库目录由属性_hive.metastore.warehouse.dir_控制，并且默认值为_/user/hive/warehouse_。所以，可以在本地文件系统的目录_/user/hive/warehouse/records_中找到对应表**records**的文件：

```
% ls /user/hive/warehouse/records/
sample.txt
```

本例中，只有一个文件_sample.txt_，通常会有更多文件，查询表时，Hive会读取表对应目录中的所有文件。

_LOAD DATA_语句中的_OVERWRITE_关键字告诉Hive，删除对应表的目录中的所有文件。如果没有_OVERWRITE_，新的文件会被简单地加入到表目录中（如果文件与表目录中文件同名，则替换已经存在的文件）。

执行查询：

```
hive> SELECT year, MAX(temperature)
    > FROM records
    > WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
    > GROUP BY year;
1949    111
1950    22
```

这个SQL查询，并不特别。特别的是，Hive把这个查询语句转换为一个job，这样，Hive基于原生数据执行了SQL查询，这就是Hive的魔力。

## 3、Comparison with Traditional Databases

尽管Hive与传统的数据库在很多方面类似（例如支持SQL接口），但是它的基础HDFS和MapReduce意味着许多架构不同直接影响了Hive支持的特性。随着时间推移，这些限制已经被渐渐移除，结果是Hive看起来用起来越来越像传统数据库。

### 3.1、Schema on Read Versus Schema on Write

在传统数据库中，在数据加载时强行执行表的schema。如果加载的数据不符合schema，会被拒绝。这种设计被叫作_**schema on write**_，因为当把数据写入数据库中时检查数据是否符合shcema。

Hive在执行查询时才验证数据，这叫作_**schema on read**_。

两种方式各有利弊。Schema on read初始加载非常快，因为不用读取、不用解析、并且不用以数据库的内部格式序列化到磁盘。Hive的加载操作只是文件复制或者移动。

Schema on write使查询时间更快，因为数据库可以对列进行索引，并且执行数据的压缩。但是，它加载数据到数据库时消耗的时间更长。另外，有许多加载数据时不知道schema的场景，使用Hive更合适。

### 3.2、Updates，Transactions，and Indexes

Updates，transactions，和indexes是传统数据库的支柱。这些特征还不是Hive的特征。因为Hive用于使用MapReduce操作HDFS，全表扫描是常态操作，表更新的目的通过将数据转换到新的表中实现。对于在大量数据集上运行的数据建仓应用，Hive表现良好。

从Hive release0.14.0开始，支持更细粒度的变化，可以调用**INSERT INTO TABLE...VALUES**插入小批量的数据。另外，也有支持**UPDATE**和**DELETE**表中行的可能。HDFS不支持更新原文件，所以，插入、更新和删除导致的变化保存在小的增量文件中。metastore定期会在后台运行了一些MapReduce jobs，把增量文件合并到基础表文件中。这些特征只在transactions的上下文中（Hive 0.13.0引入）工作，所以这些操作针对的表需要在支持transactions的基础上运行。读取表的查询被保证能够看到一个表的一致性快照。

Hive也支持表级别和分区级别的锁。例如，锁防止一个进程在另一个进程读取表的时候把表删掉。使用ZooKeeper透明地管理锁，所以用户不必获取或者释放锁，尽管可以使用**SHOW LOCKS**查看持有的锁的信息。分区的锁和表的锁是不同的，需要分别解锁；使用`unlock table <table_name>`解除锁表，使用`unlock table <table_name> partition(<partition_col>=<partition_value>)`解锁分区。默认情况下，没有启用Hive锁（设置属性`hive.support.concurrency`为`true`可以开启锁机制，`hive.zookeeper.quorum`配置Hive的表锁管理器使用的ZK服务地址）。

在某些情况下，Hive可以提升查询速度。例如，查询_SELECT \* FROM t WHERE x = a_，能够从给列x加索引受益，因为这样的话只用读取表的一小部分文件。目前有两种索引类型：_compact_和_bitmap_。（索引的实现是可插拔的，所以可以为不同的应用场景使用不同的索引实现。）

compact索引保存每个值的HDFS blocks号，而不是每个文件的偏移量，所以即使不需要很多的磁盘空间，对于值都集聚在相近的行中的情形，仍然是高效的。bitmap索引使用压缩的bitsets来高效地保存某个特定值所在的行，它们通常适合于基数（唯一值个数）较小的列（例如性别或国别）。

### 3.3、SQL-on-Hadoop Alternatives

自从有了Hive，出现了许多其它的SQL-on-Hadoop引擎以解决Hive的限制。Cloudera Impala，一个开源的交互式SQL引擎，是第一个，相比基于MapReduce运行的Hive，Impala有非常大的性能提升。Impala采用的是MPP（大范围并行处理，massive parallel processing）架构。Impala使用运行在集群中每个节点的专用daemon。当客户端运行查询时，它联系任意一个运行Impala daemon的节点，这个节点会作为查询的coordinator节点。coordinator把工作发送给集群中的其它Impala daemons，并且组合来自其它Impala daemons结果作为查询的最终完整结果。Impala使用Hive metastore并且支持Hive格式和大多数HiveQL contructs（plus SQL-92），所以，实践中，在两种系统直接迁移很简单直接，或者可以在同一集群运行两者。

Presto与Impala类似也是采用MPP架构。

Hive并非没有进步，通过支持Tez作为执行引擎，Hive也能够提升性能。

也有其它主流开源的Hive替代选项，包括Facebook的Presto，Apache Drill和Spark SQL。Presto和Drill有与Impala类似的架构，尽管Drill目标是SQL:2011而不是HiveQL。Spark SQL使用Spark作为底层引擎，可以在Spark程序中嵌入SQL查询。

注意：Spark SQL与使用Hive内部的Spark执行引擎（Hive on Spark）不同。Hive on Spark是Hive项目的一部分，支持Hive的所有特征。Spark SQL，则是一个全新的SQL引擎，具有一定级别的Hive兼容性。

Apache Phoenix使用完全不同的方式：它提供基于HBase的SQL。SQL访问通过一个JDBC driver，把查询转换为HBase扫描，并且利用HBase coprocessors的优势执行服务器侧的聚合。Metadata也保存在HBase中。

