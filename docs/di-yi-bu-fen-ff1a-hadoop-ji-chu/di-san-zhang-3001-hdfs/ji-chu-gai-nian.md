## 1、HDFS的设计（The Design of HDFS）

HDFS被设计用于大规模数据（Very Large files）可以流式数据访问，运行在商用硬件集群上。

HDFS的设计理念是最高效的数据访问类型是写一次、读多次。每次分析都涉及整个数据集的全部或者大部分，所以读取整个数据集的时间比读取某一条记录的延时要更重要。

Hadoop不需要昂贵的、高可靠性的硬件。它被设计为运行在普通商用机集群上，集群的节点失败几率很高，至少对大的集群如此。HDFS被设计为在节点失败时仍然能够对用户无打扰的正常工作。

HDFS**不适合**于低延时的时间访问（Low-latency data access）、很多小文件、多用户写、随意修改的场合。

HDFS的namenode在内存中保存文件系统的metadata，文件系统中文件的数量受限于namenode内存的容量。每个文件、目录、block大概占用150bytes。

HDFS只有一个用户写，总是在文件的末尾写，只能追加。不支持多用户写，不支持任意修改。

## 2、HDFS概念（HDFS Concepts）

### 2.1、块（Blocks）

磁盘有块大小（block size），是读写数据的最小量。HDFS也有块（Block）的概念，默认大小128M。与磁盘文件系统不同的是，HDFS中小于一个block大小的文件不会占用整个block的底层存储。

HDFS中block大小比较大的原因是减少寻道（seek）的开销。如果block足够大，传输数据消耗的时间会明显大于寻找block起始位置的开销。这样，传输由多个block组成的大文件近似于以磁盘的传输速率，而不会受制于寻道（seek）时间。

在分布式文件系统中使用block的抽象概念有几个好处：

* **第一**、也是最明显的好处是：一个文件可以比网络中任何单一磁盘大。没有任何需求要一个文件的blocks保存在同一个磁盘上，所以可以利用整个集群的磁盘。
* **第二**、用block的抽象概念而不是文件这个概念简化了存储子系统（storage subsystem）。存储子系统处理blocks，简化了存储的管理（blocks是固定大小的，很容易计算一个磁盘可以存储多少个blocks）并且消除了metadata顾虑（因为blocks只是存储数据的一块，文件metadata例如许可信息不必和block一起存储，另一个系统可以单独处理metadata）。
* **另外**，在容错和可用性方面，blocks和复本（replica）很配。为了防止block的损坏和磁盘或机器的损坏，每个block被复制到一定数量的（通常3个）物理上独立的机器。如果一个block不可用，可以从另一个位置以一种对客户端透明的方式读取一个复本。不可用的block会从它的备用位置备份到其它可用的机器以恢复到正常的备份数量。类似地，一些应用可能选择设置一个较高的block备份因子以在集群中分散读取负载。

HDFS的 _fsck_ 命令会列出文件系统中组成每个文件的blocks：

```shell
hdfs fsck / -files -blocks
```

### 2.2、Namenodes和Datanodes

HDFS集群以master-worker模式运行两种类型的nodes：一个**namenode**（_**master**_）和几个**datanodes**（_**worker**_）。namenode管理文件系统命名空间（filesystem namespace）。它维护文件系统树（filesystem tree）和文件系统树中的所有文件和目录的元数据（metadata）。这些信息以两个文件（namespace image和edit log）的形式永久地保存在本地磁盘上。namenode也知道datanodes上给定文件的所有blocks的位置；可是，它不永久保存block的位置信息，因为在系统启动时这些信息可用从datanodes重新构成。

**客户端**代表用户通过和namenode、datanodes的通信访问文件系统。客户端提供一个类似于POSIX（可移植操作系统接口）文件系统接口，用户代码不用知道namenode和datanodes如何工作。  
DataNodes是文件系统的主力。它们存储和获取blocks，并且定期向namenode报告它们存储的blocks的列表。

没有namenode，文件系统就不能用。事实上，如果运行namenode的机器崩溃，文件系统中的所有的文件都会丢失，因为无从知晓怎样从datanodes的blocks重新组成文件。因此，使namenode能够从失败复原很重要。Hadoop提供了两种恢复机制：

* 第一种方式是：**备份**组成文件系统元数据持久状态的文件。通过配置可以让Hadoop把它的持久状态写到多个文件系统中。这些些操作是同步的、原子的。通常的配置是写到本地磁盘和远程的NFS。
* 也可以运行一个**第二namenode**（_secondary namenode_），尽管不是namenode。它的主要角色是定期的将namespace image和edit log合并，以防止edit log变得太大。第二namenode通常运行在一个独立的物理机上因为它需要足够的CPU以及和namenode同样多的内存来进行合并处理。它保持着一个合并过的namespace image，可以在namenode崩溃的时候使用。可是，第二namenode的状态要比主namenode落后，当主namenode完全崩溃时，肯定会有数据的丢失。遇到这种情况，通常的方法是把NFS上备份的metadata文件复制到第二namenode，并把它作为主namenode运行。（注意，也可以运行一个热的待用（_hot standby_）namenode来代替第二namenode）

### 2.3、块缓存（Block Caching）

通常datanode从磁盘读取blocks，但是对于频繁访问的文件blocks可以被明确的保存到datanode的内存中，以非堆block cache的形式。默认情况下，一个block只会缓存到一个datanode的内存中，尽管可以针对每个文件配置这个数量。Job调度器（对MapReduce、Spark、等）可以通过在缓存block的datanode上运行tasks来享受缓存blocks带来的优势，从而提高读取效果。例如，一个用在join操作中的小的查找表（lookup table）是使用block 缓存的好的候选者。

用户或者应用通过添加_**cache directive**_到_**cache pool**_来指示namenode缓存那些文件（缓存多久）。cache pools是用于管理缓存权限和资源使用的管理分组。

### 2.4、HDFS联盟（HDFS Federation）

对于一个有非常多文件的非常大的集群，namenode的内存会成为集群存储数据扩展的瓶颈。在2.x发布版本中，引入HDFS联盟，允许一个集群通过添加namenodes进行扩展，每个namnode管理文件系统namespace的一部分。例如，一个namenode管理目录/user中的所有文件，另一个namenode管理/share目录下的文件。

在HDFS联盟中，每个namenode**管理**一个_**namespace volume**_（命名空间卷），每个namespace volume有namespace的metadata和包括namespace中文件的所有blocks的一个_**block pool**_组成。namespace volume之间相互独立，这意味着namenodes之间没有通信，此外一个namenode的崩溃不会影响其它namenodes管理的namespace的可用性。**Block pool存储是不分区的**，可是，这样datanodes可以注册到集群中的每个namenode并存储多个**block pool**的blocks。

要访问联盟的HDFS集群，客户端使用客户端侧（client-side）挂载表（mount tables）来映射namenodes中的文件路径。通过配置使用_**ViewFileSystem**_和_viewfs://URIs_来管理

### 2.5、HDFS高可用（HDFS High Availability）

namenode的两种恢复机制不能满足高可用的要求，完全恢复namenode，要实现以下三点：

* 1、加载namespace到内存中
* 2、重播（replay）edit log
* 3、从datanodes获取足够的block reports来离开安全模式（safe mode）

在有许多文件和blocks的大集群中，恢复namenode到可用需要30分钟甚至更多。

对常规维护来说长的恢复时间也是一个难题。事实上，namenode的故障是很少见的，计划停机时间在实践中是很重要的。

Hadoop 2通过增加支持HDFS 高可用（HA）来补救这种情况。在实现上，有一对配置为活跃等待（_**active-standby**_）的namenodes。在活跃namenode故障的时候，等待的namenode接管活跃namenode的职责（没有明显得中断）继续服务客户端的请求。要实现高可用，需要进行一些架构的改动（architectural changes）：

* **Namenodes**必须使用高可用**共享存储**以共享edit log。当等待的namenode启动时，它读取整个的共享edit log以实现他的状态和活跃的namenode同步，然后当活跃namenode继续写edit log时，接着读取新的edit log。
* **Datanodes**必须向所有的namenodes（active or standby）发送blocks reports，因为block映射存储在namenode的内存中而不是磁盘上。
* **客户端**必须配置为能够使用对用户透明的机制来应对namenode的故障。
* **第二namenode**的角色被归入standby，对活跃的namenode的namespace进行定期的检查。

高可用的共享存储有两种选择：_**NFS文件管理器**_（NFS filer）、_**QJM**_（quorum jornal manager，_仲裁日志管理？？法定数目日志管理？_）。QJM是一个专用的HDFS实现，设计的唯一目的就是提供高可用的edit log，也是大多数HDFS安装的推荐选择。QJM运行一组_**journal nodes**_，每次编辑必须写到大多数的journal nodes。一般有三个journal nodes，所以系统可以融入其中某一个的故障。这种安排和ZooKeeper的工作方式类似，景观QJM的实现没有使用ZooKeeper。（**Note.**但是HDFS HA使用了ZooKeeper来选择活跃的namenode）

如果活跃的namenode故障，standby namenode可以很快的接管namenode（在几十秒内）因为它保存着可用的最新的状态：最新的edit log和最新的block映射。实际观测的故障排除时间会长一些（一分钟左右），因为系统需要在确定namenode失败时需要保守点儿。

在不大可能出现的活跃namenode和standby namenode都故障的情况下，管理员也能从零启动standby namenode。这不会比没有非高可用（non-HA）情况更坏，从操作的方面来说这是一个进步，因为这个过程是Hadoop内置的标准流程。

### 2.6、故障转移和防护（Failover and fencing）

从活跃namenode到standby namenode的转移是由系统中叫做_**failover controller**_的新的实体管理的。有不同的failover controllers，但是默认的实现是使用ZooKeeper来确保只有一个namenode是活跃的。每个namenode运行一个轻量级的failover controller进程，它的作用是（使用简单的心跳机制）监控它的namenode的故障并当namenode故障时触发故障转移。

故障转移也可以由管理员手动进行，例如，在常规维护的情况下。这叫做**优雅的故障转移**（graceful failover），因为failover controller安排一个有序地转换来切换两种namenodes的角色。

在非优雅故障转移的情况下，可是，要确定故障的namenode已经停止运行是不可能的。例如，缓慢的网络和网络分区会触发故障转移，尽管之前的活跃namenode仍然在运行并仍然把它作为活跃namenode。HA实现在确保之前活跃的namenode不会造成损害做了很大的努力——一个_**fencing**_方法。

QJM一次只允许一个namenode来写edit log；可是，之前活跃的namenode仍然有可能为旧的客户端请求提供服务，所以，建立一个可以终止namenode进程的SSH防护命令是一个好的想法。当使用NFS文件管理器（NFS filer）共享edit log时，需要更强的防护方法，因为对它来说，一次只允许一个namenode写edit log是不可能的（这也是推荐使用QJM的原因）。**防护机制的范围**包括撤销namenode访问共享存储目录的权限（一般使用一个供应商特定&lt;vendor-specific&gt;NFS命令）和废除它的网络端口（通过远程管理命令）。最后一招是，直接物理机断电。

**客户端故障转移**由客户端库透明地处理。最简单的实现是使用客户端侧（client-side）配置来控制故障转移。HDFS URI使用了一个逻辑主机名（logical hostname）映射到一对namenode地址（在配置文件中），客户端库尝试每个namenode地址直到操作成功。

## 3、命令行接口（The Command-Line Interface\)

* pseudodistributed mode伪分布模式

```shell
hdfs dfsadmin -report
```

运行结果：

```
Configured Capacity: 496885559296 (462.76 GB)
Present Capacity: 486843913918 (453.41 GB)
DFS Remaining: 486843912192 (453.41 GB)
DFS Used: 1726 (1.69 KB)
DFS Used%: 0.00%
Under replicated blocks: 4
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (1):

Name: 127.0.0.1:50010 (127.0.0.1)
Hostname: Millor
Decommission Status : Normal
Configured Capacity: 496885559296 (462.76 GB)
DFS Used: 1726 (1.69 KB)
Non DFS Used: 10041645378 (9.35 GB)
DFS Remaining: 486843912192 (453.41 GB)
DFS Used%: 0.00%
DFS Remaining%: 97.98%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Aug 31 17:35:43 CST 2019
```

### 3.1、基本文件系统操作（Basic Filesystem Operations）

基本文件系统操作命令_**hadoop fs**_
_**hadoop fs**_命令支持与HDFS交互，也支持和其他Hadoop支持文件系统（比如本地文件系统（Local FS），WebHDFS，S3 FS等）交互。

※ 相似的命令_**hadoop dfs**_、_**hdfs dfs**_，只能操作HDFS文件系统（包括与Local FS间的操作），前者已经Deprecated，一般使用后者。[官方文档](http://hadoop.apache.org/docs/r1.0.4/cn/hdfs_shell.html)

命令行语法：
```shell
hadoop fs [generic options，通用选项] [command options，命令选项]
```
应用举例：
```shell
# 查看hadoop fs命令的帮助
hadoop fs -help


# 把本地文件复制到HDFS
hadoop fs -copyFromLocal input/docs/quangle.txt \
hdfs://localhost/user/tom/quangle.txt

# 可以省略shcema和host部分，使用默认的地址，即core-site.xml中指定的地址，
# 如果core-site.xml指定为hdfs://localhost，则下面命令和上面一样
hadoop fs -copyFromLocal input/docs/quangle.txt /user/tom/quangle.txt

# 也可以使用相对路径，复制文件到HDFS的home目录
hadoop fs -copyFromLocal input/docs/quangle.txt quangle.txt

# 把文件从HDFS复制到本地文件系统
hadoop fs -copyToLocal quangle.txt quangle.copy.txt

# 在HDFS上创建目录
hadoop fs -mkdir books

# 查看当前所处HDFS目录
hadoop fs -ls .

# 查看HDFS的根目录
hadoop fs -ls /
```

_hadoop fs -ls &lt;dir&gt;_命令的使用方式，与Linux的_ls_命令类似。

运行结果：

```shell
drwxr-xr-x   - root root          0 2017-09-11 04:05 basedata_win_type
-rw-r--r--   3 root root       5314 2017-11-06 17:23 huawei.csv
-rw-r--r--   3 root root       4428 2017-11-06 17:36 huawei.txt
```

第一列，文件模式；

第二列，复本数量（目录没有复本的概念，目录被作为元数据保存在namenode，而不是datanodes）

第三、四列，显示文件拥有者和分组

第五列，文件字节大小，目录为0

第六、七列，最后修改日期和时间

第八列，文件/目录名称。

HDFS文件/目录有三类权限：读（r）、写（w）、执行（x）。

对文件来说执行权限是被忽略的，因为，在HDFS上是不能执行文件的。对目录来说执行权限是它的子元素的访问权限。

默认情况下Hadoop的安全（security）是失效的，意味着客户端可以访问HDFS而不需要授权。但是不能让任何人随意修改HDFS，所以HDFS的权限管理还是有必要的。可以通过_dfs.premissions.enabled_来开启或者关闭权限验证，默认是开启的。还有一个超级用户（superuser）的概念，namenode就是超级用户，权限检查不限制超级用户。

## 4、Hadoop文件系统（Hadoop Filesystems）

Hadoop有文件系统的抽闲概念，HDFS只是一种实现。Java抽象类org.apache.hadoop.fs.FileSystem代表Hadoop中一个文件系统的客户端接口，它有一些具体实现，主要如下：

###### _Table 3-1.Hadoop filesystems_

| **Filesystem** | **URI scheme** | **Java implementation**（都在org.apache.hadoop包中） | 描述 |
| :--- | :--- | :--- | :--- |
| **Local** | file | fs.LocalFileSystem |  |
| **HDFS** | hdfs | hdfs.DistributedFileSystem |  |
| **WebHDFS** | webhdfs | hdfs.web.WebHdfsFileSystem | 提供基于HTTP的需要认证的读写访问的文件系统 |
| **Secure WebHDFS** | swebhdfs | hdfs.web.SWebHdfsFileSystem | WebHDFS的HTTPS版本 |
| **S3** | s3a | fs.s3a.S3AFileSystem | 基于Amazon S3的问句系统 |
| **Azure** | wasb | fs.azure.NativeAzureFileSystem | 基于Microsoft Azure的文件系统 |

Hadoop提供了许多访问它的文件系统的接口，通常使用URI scheme来获取对应的文件系统实例来进行通信。例如，查看本地文件系统根目录：

```
hadoop fs -ls file:///
```

尽管可以运行MapReduce程序访问任何Hadoop支持的文件系统，但是当处理大规模数据的时候还是应该选择由数据本地性优化的分布式文件系统，特别是HDFS。

### 4.1、接口（Interfaces）

Hadoop是用Java编写的，所以大多数的Hadoop文件系统交互是通过Java API进行的。例如，文件系统shell，就是Java应用，使用Java的_FileSystem_这个类提供文件系统操作。这些接口最常用的是和HDFS进行交互。其它文件系统由其它对应的工具。

### 4.2、HTTP

用HTTP REST API和HDFS交互，比Java客户端要慢。

### 4.3、C

Hadoop由C API。通常比Java版本API滞后。

### 4.4、NFS

使用Hadoop的NFSv3网关可以将HDFS挂载到本地客户端的文件系统。

### 4.5、FUSE

filesystem in userspace（FUSE），把HDFS挂载为一个标准的本地文件系统。

