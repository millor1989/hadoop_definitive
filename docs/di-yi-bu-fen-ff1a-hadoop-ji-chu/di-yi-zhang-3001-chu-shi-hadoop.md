# Chapter 1. Meet Hadoop

初识Haoop

## 1、数据！（Data!）

数据爆炸的时代，面临数据存储和分析的问题。

## 2、数据存储和分析（Data Storage and Analysis）

对于磁盘，尽管容量提升很快，但是传输速率（transfer speed、读取速率 access speed）提升相对很慢；由于单个磁盘容量和速度的问题，大量数据读写可以通过并行读取多个磁盘解决。但是面临两个问题：

1、硬件故障问题：解决途径备份

2、综合分析各个磁盘读取的数据的问题：解决方法MapReduce

Hadoop是一个用于数据存储分析的可靠、可扩展平台。运行在商用硬件上、开源，因而实惠。

## 3、查询所有的数据（Querying All Your Data）

MapReduce是一个变革性的（transformative）批查询处理器（batch query processor），能够在整个数据集上执行特定的查询并且在合理的时间内取得结果。

## 4、批处理之外（Beyond Batch）

MapReduce本质上是一个批处理系统，不适用于交互式分析。不能通过它在几秒钟之内获得查询结果。查询一般要耗时几分钟甚至更多，它适用于离线使用。

Hadoop已经从它的原始的化身，演变得已经超越了批处理。“Hadoop”有时指的是一个大型的包含许多项目的生态系统，不仅仅HDFS和MapReduce这些分布式运算和大规模数据处理的基础设施。也包含了一些其它项目，例如：

**HBase：**第一个提供在线访问的组件，一个使用HDFS作为底层存储的key-value存储系统。HBase提供了在线读写个别行的途径和读写批量数据的批操作。

**YARN：**Hadoop 2中YARN的引入推动了Hadoop中新的处理模式。YARN是一个集群资源管理系统，允许分布式应用（不仅仅MapReduce）在Hadoop集群的数据上运行。

还有一些与Hadoop协同工作的其它类型的项目：

**交互式SQL（Interactive SQL）**

通过使用MapReduce分发并且使用分布式查询引擎（**Impala**使用专用“always on”守护线程daemons；基于Tez的Hive使用容器重用_（container reuse）_），使从SQL查询获取低延时的响应成为可能，尽管查询的是大规模的数据集。

**迭代处理（Iterative processing）**

许多算法（像机器学习算法）本质上是迭代的，所以和每次迭代都从磁盘加载数据相比，把每次的中间工作集（intermediate working set）保存在内存中是非常高效的。例如Spark。

**流处理（Stream processing）**

像Strom、Spark Streaming、Samza这些流系统，能够在数据流上进行实时的、分布式的运算并且把结果发送到Hadoop存储中或外部系统。

**查询（Search）**

Solr查询平台可以在Hadoop集群上运行，在文档被添加到HDFS的时候进行索引，并通过存放在HDFS中的索引提供查询服务。

尽管Hadoop上出现了不同的处理框架，MapReduce仍然占有批处理的一席之地，因为它拥有应用广泛的一些理念（输入格式、数据集分割），理解它的工作原理还是很有用的。

## 5、和其它系统的比较（Comparison with Other Systems）

Hadoop不是第一个用于数据存储和分析的分布式系统，但是一些特性让它和其它的类似系统显得不同。

### 5.1、关系型数据库管理系统（Relaitional Database Management Systems）

为什么不用有很多磁盘的数据库进行大规模的分析？为什么需要Hadoop？

对于磁盘驱动器来说，寻道时间（seek time）的提升比传输速率慢。寻道是指把磁盘探头移动到磁盘的一个指定的地方进行数据读写。寻道时间对应于磁盘操作的延时，而传输速率对应于磁盘的带宽。

如果数据访问类型由寻道控制，读写数据集大部分消耗的时间要比传输数据（streaming through it，operates at the transfer rate）更长。另一方面，要更新数据库一小部分的记录，传统的B-Tree（管系型数据库的数据结构，受限于寻道速率）表现良好；如果要更新数据库的大部分，B-Tree就没有MapReduce高效了，MapReduce使用Sort/Merge来重构数据库。

在许多方面，MapReduce可以看作是关系型数据管理系统的一个补充。MapReduce适合于以批量形式分析整个数据集，特别是专门的分析。RDBMS适用于点查询或更新，数据集被索引以降低获取和更新相对少量数据的时间。MapReduce适用于数据只写一次读取多次的应用，而RDBMS适用于数据集持续更新的应用。

_**表1-1、RDBMS和MapReduce比较（RDBMS compared to MapReduce）**_

|  | Traditional RDBMS | MapReduce |
| :--- | :--- | :--- |
| **Data size** | Gigabytes | Petabytes |
| **Access** | Interactive and batch | Batch |
| **Updates** | Read and write many times | Write once，read many times |
| **Transactions** | ACID | None |
| **Structure** | Schema-on-write | Schema-on-read |
| **Integrity** | High | Low |
| **Scaling** | Nonlinear | Linear |

但是，关系型数据库和Hadoop系统的区别正在变得模糊。关系型数据库已经开始包括一些Hadoop的想法，另一方面，Hadoop系统例如Hive变得更加具有交互性（通过远离MapReduce）并增加了像索引和事务特征让它看起来越来越像传统的RDBMS。

另一个Hadoop和RDBMS的不同是数据集的结构：

**结构化数据（Structured data）**有预定义的格式，例如XML文档或者数据表符合特定的预定义的模式（schema）。

**半结构化数据（Semi-structured data\)**，是宽松的，尽管有一个模式（schema），常常忽视格式，格式可能只作为数据结构的向导。例如，电子表格（spreadsheet），结构是单元格组成的表，而不管单元格里面的数据格式。

**非结构化数据（Unstructured data）**没有任何特定的内部结构，例如，text和图片数据。

Hadoop在非结构化、半结构化数据上运行良好因为它被设计为在处理时才解析数据（Schema-on-read）。这提供了灵活性并且避免了像RDBMS那样的高成本的数据加载阶段，因为Hadoop只是一个文件copy。

关系型数据通常是**标准化**（_normalized_）的以维护他的完整性并移除冗余（redundancy）。标准化会给Hadoop处理带来问题，因为它会使读取一条记录变为一个非本地的操作，而Hadoop的一个核心假设是可以进行（高速的）流式读写。

Web server log是一个非标准化记录的例子，这也是Hadoop很适合分析log文件的一个原因。Hadoop可以进行join，但是用的没有关系型数据库多。

MapReduce——和Hadoop中的其它处理模型——可以随着数据大小线性地扩展。数据是分区的，功能原语（functional primitives）（像map和reduce）可以在分割的分区上并行进行。这意味着，如果数据量加倍，job运行速度减半。但是如果集群的运算能力也加倍，job运行速度不会变慢。但是RDBMS的SQL查询不是线性的。

### 5.2、网格运算（Grid Computing）

高效运算（HPC，high-perfomance computing）和网格运算（grid computing）社区已经用这种应用程序接口（API，application program interfaces）作为消息传输接口（MPI，Message Passing Interface）做了多年的大规模数据处理。笼统的讲，HPC的方法是在集群的机器中分发工作，访问存储区域网络（SAN，storage area netowork）中一个共享的文件系统。这主要对计算密集型（compute-intensive）jobs工作良好，但是当需要访问大规模的数据（数百G）时就成为一个问题，因为网络带宽成为瓶颈，而计算节点变得空闲。

Hadoop尝试把计算节点和本地数据结合，这样数据访问就很快因为数据就在本地。这个特征就是**数据本地性（**_**data locality**_**）**，它是Hadoop数据处理的核心也是Hadoop良好表现的原因。网络带宽是数据中心的最宝贵资源，Hadoop通过明确地建模网络拓扑（modeling network topology）尽量节省网络带宽。注意，Haoop中这中安排不排除高CPU分析。

MPI对开发者要求很高，要处理底层很多东西；Hadoop中的处理只在操作高级别：开发者只用考虑数据模型，不用管底层的数据流。

协调大规模分布式运算中的进程是一个挑战。最难的方面是优雅的处理部分失败——当不知道一个远程进程是否失败——仍然能够让总体运算进行。像MapReduce这种分布式处理框架，让程序员免于必须考虑进程的失败，因为它能够检测失败的tasks并且在健康的设上重启task。MapReduce能够这样做是因为他是一个不共享任何数据（shared-nothing）框架，tasks之间没有依赖（这有点过分简述了，因为mappers的输出给到了reducers，但是这是在MapReduce系统的控制下；这种情况下，需要给予重启失败的reducer比重启失败的mapper更多的关注，因为要确保它能够获取必要的map输出，如果没有，就要通过重新运行相关的mapper来重新生成输出文件）。从开发者的角度来看，怎样重新运行task是不用关心的。但是MPI则不同，需要开发者处理检查点和恢复等操作，虽然给了开发者控制权，但是使开发变得困难。

### 5.3、志愿者运算（Volunteer Computing）

志愿者贡献计算能力，得到分发的工作单元（work unit），返还计算结果。为了防止欺诈，每个工作单元被发送给三个不同的志愿者，需要至少其中两个结果一致。

## 6、Hadoop简史（A Brief History of Apache Hadoop）

起源于Apache Nutch（Apache Lucene的一部分），2002年，有Doug Cutting创建，Hadoop名字来源于他女儿的玩具象。

2003受Google的GFS影响，于2004年完成了Nutch Distributed Filesystem（NDFS）。

2004年Google发表了MapReduce论文。2005年MapReduce有了Nutch实现版本。

2006年将NDFS和MapReduce实现版本独立为Hadoop项目。同年Doug Cutting加入Yahoo!。

2008年2月Yahoo!宣布它的产品查询索引是由10000核心的Hadoop集群产生的。



























