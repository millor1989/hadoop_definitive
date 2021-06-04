## 3、Hadoop Configuration

有些控制Hadoop安装配置的文件。**表10-1**列出了最重要的配置文件。

###### 表 10-1. Hadoop configuration files

| Filename | Format | Description |
| --- | --- | --- |
| _hadoop-env.sh_ | bash脚本 | 脚本中使用的用于运行Hadoop的环境变量 |
| _mapred-env.sh_ | bash脚本 | 脚本中使用的用于运行MapReduce的环境变量（覆盖_hadoop-env.sh_脚本中的变量） |
| _yarn-env.sh_ | bash脚本 | 脚本中使用的用于运行YARN的环境变量（覆盖_hadoop-env.sh_脚本中的变量） |
| _core-site.xml_ | Hadoop配置XML | Hadoop Core的配置设置，例如HDFS、MapReduce、YARN的通用I/O设置 |
| _hdfs-site.xml_ | Hadoop配置XML | HDFS daemons：namenode、次要namenode和datanodes的配置设置 |
| _mapred-site.xml_ | Hadoop配置XML | MapReduce daemons：job history server的配置设置 |
| _yarn-site.xml_ | Hadoop配置XML | YARN daemons：资源管理器、web app proxy server、节点管理器的配置设置 |
| _slaves_ | 普通文本 | 运行datanode和节点管理器的机器集合（一行一个） |
| _hadoop-metrics2.properties_ | Java properties | 控制Hadoop中metrics如何展示的属性 |
| _log4j.properties_ | Java properties | 系统日志文件、namenode audit日志、task JVM进程的task日志属性 |
| _hadoop-policy.xml_ | Hadoop配置XML | 以Hadoop安全模式运行时，访问控制配置设置集合 |

这些文件都在Hadoop发布版本的_etc/hadoop_目录中。只要启动daemons时使用_**--conf**_选项（或者，等价地，使用环境变量**HADOOP\_CONF\_DIR**设置）指定了本地文件系统中配置目录的位置，配置目录可以改为文件系统的其它目录（放置Hadoop安装目录外，让升级稍微容易一点儿）。

### 3.1、Configuration Management

Hadoop没有单一、全局的配置信息。机器中的每个Hadoop节点都有自己的配置文件，它们由管理员决定以确保它们是整个集群同步的。并行shell工具（_dsh_、_pdsh_）可以做到这点。这是如Cloudera Manager和Apache Ambari这样的Hadoop集群管理工具比较擅长的领域，因为它们能够处理整个集群配置变化的传播。

Hadoop被设计为以便可以使用仅仅一套配置文件，这些配置文件可以用于所有的master和worker机器。这样做的最大优势是简单，概念上（只需要处理一个配置）和操作上（Hadoop脚本能高效处理一个配置设置）都简单。

对某些集群，一刀切（one-size-fits-all）的配置模式失效了。例如，如果用不同于集群现有硬件规格的新机器进行集群扩展，需要对新机器使用不同配置以利用新机器的资源。

在这些情况下，需要有机器的类（_class_）概念并且为每个类维护不同的配置。Hadoop没有提供这种作用的工具，但是有些精确管理这种类型配置的优秀工具可以使用，例如Chef，Puppet，CFEngine，Bcfg2。

对于任何大小的集群，保持所有的机器同步都是一个挑战。例如，进行更新数据的时候，机器恰好故障，在机器恢复时要确保数据更新。这是一个大问题，并且会导致歧义安装（divergent installations），所以，即使使用Hadoop控制脚本管理Hadoop，使用维护集群的配置管理工具可能会是一个好的想法。这些工具在进行常规维护（比如修补安全漏洞和更新系统包）方面也很优秀。

### 3.2、Environment Settings

#### 3.2.1、Java

使用的Java环境由_hadoop-env.sh_中设置的**JAVA\_HOME**决定，或者，如果_hadoop-env.sh_没有设置，则由shell环境变量**JAVA\_HOME**决定。最好在_hadoop-env.sh_中设置，以便在一个地方明确定义，并且确保整个集群使用相同版本的Java。

#### 3.2.2、内存堆大小（Memory heap size）

默认情况下，Hadoop为它运行的每个daemon分配1GB的内存。这由_hadoop-env.sh_中的**HADOOP\_HEAPSIZE**设置决定。也有可以改变某一个daemon堆大小的环境变量，例如，可以设置_yarn-env.sh_中的**YARN\_RESOURCEMANAGER\_HEAPSEIZE**来覆盖资源管理器的堆大小。

令人惊讶的是，没有对应HDFS daemons堆内存的环境变量。但是，通常给namenode更多的堆内存。

##### 3.2.2.1、the memory namenode need

namenode可以吃光内存，因为每个文件的每个block的引用是在内存中维护的。namenode使用的内存由每个文件blocks个数、文件名长度、文件系统中的目录数决定；并且每个Hadoop版本也可能不同；所以很难精确计算。默认的1GB namenode内存一般对于数百万的文件是足够的，但是，作为namenode内存调整的经验之谈（a rule of thumb），保守地，每百万blocks的存储对应1GB namenode内存。

通过设置_hadoop-env.sh_中的**HADOOP\_NAMENODE\_OPTS**为包含一个设置内存大小的JVM选项的值，可以在不改变分配给其它Hadoop daemons的内存的情况下增加namenode的内存。**HADOOP\_NAMENODE\_OPTS**可以向namenode的JVM传递额外的选项，例如，如果过在使用一个Sun JVM，_**-Xmx2000m**_能够指定应该给namenode分配2000MB的内存。

如果改变了namenode的内存分配，要记得同样地改变次要namenode（使用变量**HADOOP\_SECONDARYNAMENODE\_OPTS**设置），因为次要namenode的内存需求与主namenode相当。

#### 3.2.3、系统日志文件（System logfiles）

Hadoop产生的系统日志文件默认保存在_$HADOOP\_HOME/logs_目录。使用_hadoop-env.sh_中的**HADOOP\_LOG\_DIR**可以改变这个位置。最好要将这个目录改为Hadoop安装目录以外的地方。改变这个配置，可以将日志文件保存到一个地方，即使因为升级安装目录改变，这个位置也不会变。常用选择是_/var/log/hadoop_，通过_hadoop-env.sh_中的如下行设置：

```
export HADOOP_LOG_DIR=/var/log/hadoop
```

如果不存在指定的目录，会自动创建。（如果不存在，要确保相关的Unix Hadoop用户有创建目录的权限。）机器上运行的每个Hadoop daemon产生两个日志文件。第一个是通过log4j输出的日志文件。这个文件名称后缀是_.log_，诊断问题时，首先要查看的就是这个文件，因为大多数应用日志信息都写在这里面。标准的Hadoop log4j配置使用按天滚动的文件appender来轮转（产生新的文件）日志文件。老的日志文件不会被删除，所以要定期对它们进行删除或者归档，以便不会耗尽本地节点的硬盘空间。

第二个日志文件是联合的标准输出和标准错误日志。这个日志文件，名称后缀为_.out_，通常包含很少或者没有输出，因为Hadoop用log4j来记录日志。只有在daemon重启时才轮转，并且只保存最近的5个日志文件。旧的日志文件有一个从1到5的后缀数字，带5的是最老的文件。

日志文件名（两种类型）都是运行daemon的用户名，daemon名，和主机名的组合。例如，_hadoop-hdfsdatanode-ip-10-45-174-112.log.2014-09-20_是轮转后的日志文件的名字。这种命名结构使得可以将集群中所有机器的日志文件放在同一个目录中，因为文件名是唯一的。

事实上，日志文件名中的用户名为_hadoop-env.sh_文件中**HADOOP\_IDENT\_STRING**配置的默认值。如果为了给日志文件命名的目的，想要使用不同的identity，可以将**HADOOP\_IDENT\_STRING**该为想要使用的identifier。

#### 3.2.4、SSH设置（SSH settings）

使用SSH可以在master运行远程worker节点上的命令。自定义SSH设置很有用，例如，想要减少连接超时（使用**ConnectTimeout**选项）以便控制脚本不会挂起一直等待故障节点的回应。明显地，这可能太过头（this can be taken too far）。如果超时时间太小，会跳过忙碌的节点，这也不好。

另一个有用的SSH设置是**StrictHostKeyChecking**，可以设置为_**no**_来自动地把新的host keys加到已知的hosts文件中。默认值为_**ask**_，提示用户确认key fingerprint已经通过校验，这不是大集群环境下的恰当设置。

要向SSH传递额外的选项，在_hadoop-env.sh_中定义**HADOOP\_SSH\_OPTS**环境变量。可以从ssh和ssh\_config的manual页面查看更多的SSH设置。

#### 3.2.5、Important Hadoop Daemon Properties

要查看一个运行daemon的实际配置，访问它的web server的_/conf_页面。例如，[http://resource-manager-host:8088/conf](http://resource-manager-host:8088/conf)展示了资源管理器运行时使用的配置。这个页面展示daemon运行使用的组合的site和default配置文件，也展示了每个属性来自哪个文件。

###### 例 10-1.A typical core-site.xml configuration file

```xml
<?xml version="1.0"?>
<!-- core-site.xml -->
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://namenode/</value>
  </property>
</configuration>
```

###### 例 10-2.A typical hdfs-site.xml configuration file

```xml
<?xml version="1.0"?>
<!-- hdfs-site.xml -->
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/disk1/hdfs/name,/remote/hdfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/disk1/hdfs/data,/disk2/hdfs/data</value>
  </property>

  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>/disk1/hdfs/namesecondary,/disk2/hdfs/namesecondary</value>
  </property>
</configuration>
```

###### 例 10-3.A typical yarn-site.xml configuration file

```xml
<?xml version="1.0"?>
<!-- yarn-site.xml -->
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>resourcemanager</value>
  </property>
  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>/disk1/nm-local-dir,/disk2/nm-local-dir</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce.shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>16384</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>16</value>
  </property>
</configuration>
```

#### 3.2.6、HDFS

要运行HDFS，需要指定一个机器作为namenode。这种情况下，属性_fs.defaultFS_是一个HDFS文件系统URI，它的host是namenode的主机名或者IP地址，它的端口是namenode RPCs监听的端口。如果不指定端口，默认使用8020端口。

_fs.defaultFS_属性还兼有指定默认文件系统的功能。默认文件系统用来解析相对路径，这很方便，因为这能接生编码（并且避免硬编码某个namenode的地址）。例如，**例 10-1**指定的默认文件系统，相对路径URI_/a/b_会被解析为_hdfs://namenode/a/b_。

注意，如果在运行HDFS，使用_fs.defaultFS_来指定HDFS namenode和默认的文件系统意味着：HDFS必须是服务器配置中的默认文件系统。但是，要记住，为了方便，在客户端配置中，有可能指定不同的文件系统作为默认文件系统。例如，如果使用了HFDS和S3文件系统，那么可以指定任何一个作为客户端配置中的默认文件系统，这样，既可以使用相对路径URI指向默认文件系统又可以使用绝对路径URI指向其它文件系统。

还有些其它的要为HDFS设置的配置：那些设置namenode和datanodes存储目录的配置。属性_dfs.namenode.name.dir_指定namenode保存持久化文件系统元数据（edit log和文件系统image）的目录的集合。每个元数据文件都有一个拷贝保存在每个指定的目录中。通常配置_dfs.namenode.name.dir_为能够将namenode元数据写到一个或者两个本地硬盘，和一个远程硬盘，例如一个NFS挂载目录。这种设置，保证了一个本地硬盘故障或者整个namenode故障，因为这两种情况下还能从NFS目录恢复并启动一个新的namenode。（次要namenode只是周期性检查namenode，所以它不提供namenode的最新备份。）

也要设置_dfs.datanode.data.dir_属性，它为datanode指定保存blocks的目录集合。与namenode目录设置功能不同的是，datanode在这些存储目录直接轮换的写数据，所以，为了性能，应该为每个本地磁盘指定一个存储目录。使用多个磁盘存储数据对也使读取性能受益，因为blocks会在这些磁盘中分布，并且不同blocks的同时读取会相应的扩展到这些磁盘上。

提示，为了最佳的性能，应该使用_**noatime**_选项挂载存储磁盘。这个设置意味着在读取文件时不会记录最近的访问时间信息，这能够显著的提升性能。

最后，应该配置次要namenode保存文件系统的它的检查点（checkpoints）。_dfs.namenode.checkpoint.dir_属性指定保存检查点的目录的集合。与namenode目录设置类似，检查过的文件文件系统image会保存到每个目录中。

###### 表 10-2.Important HDFS daemon properties

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| fs.defaultFS | URI | file:/// | 默认文件系统。这个URI指定了namenode的RPC服务器运行的主机名和端口。默认端口是8020，这个属性在_core-site.xml_中设置。 |
| dfs.namenode.name.dir | 逗号分隔的目录名 | file://${hadoop.tmp.dir}/dfs/name | namenode保存持久化元数据的目录集合。namenode把元数据拷贝保存到目录集合中的每个目录。 |
| dfs.datanode.data.dir | 逗号分隔的目录名 | file://${hadoop.tmp.dir}/dfs/data | datanode保存blocks的目录集合。每个block会保存在这些目录中的某一个里面。 |
| dfs.namenode.checkpoint.dir | 逗号分隔的目录名 | file://${hadoop.tmp.dir}/dfs/namesecondary | 次要namenode保存检查点的目录集合。把检查点拷贝保存在集合中的每个目录中。 |

警告，默认情况下HDFS的存储目录在Hadoop的临时目录下（临时目录通过属性_hadoop.tmp.dir_配置，默认为_**/tmp/hadoop-${user.name}**_）。所以，关键是，这些目录应该设置为在系统清理临时目录时，系统不会丢失这些数据。

#### 3.2.7、YARN

要运行YARN，需要指定一台机器作为资源管理器。最简单的方式是设置属性_yarn.resourcemanager.hostname_为运行资源管理器的机器的主机名或者IP地址。许多资源管理器服务的地址都是从这个属性派生的。例如，_yarn.resourcemanager.address_的格式为host-port对，host默认为_yarn.resourcemanager.hostname_。在一个MapReduce客户配置中，这个属性用于通过RPC连接到资源管理器。

在一个MapReduce job中，中间数据和工作文件写到临时的本地文件。因为这个数据包含潜在性的非常大的map task的输出，要确保_yarn.nodemanager.local-dirs_属性（控制YARN 容器的本地临时存储位置）使用的是足够大的磁盘分区。这个属性接收逗号分隔的目录名集合，应该使用所有可用的本地磁盘以展开磁盘I/O（这些目录以轮换的方式使用）。一般，YARN本地存储使用与datanode block存储相同的磁盘和分区（但是不同的目录）。

与MapReduce 1不同，YARN没有tasktrackers为reduce task提供map输出，所以这个功能依赖混洗handlers，它们是在节点管理器中长时间运行的辅助服务。因为YARN是一个通用服务，需要在_yarn-site.xml_文件中通过设置_yarn.nodemanager.aux-services_属性为**mapreduce\_shuffle**来开启MapReduce混洗handlers。

###### 表 10-3.Important YARN daemon properties

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| yarn.resourcemanager.hostname | Hostname | 0.0.0.0 | 资源管理器运行的机器的主机名。以下简称为${y.rm.hostname} |
| yarn.resourcemanager.address | Hostname and port | ${y.rm.hostname}:8032 | 资源管理器的RPC服务运行在的hostname和port |
| yarn.nodemanager.local-dirs | 逗号分隔的目录名称 | ${hadoop.tmp.dir}/nm-local-dir | 节点管理器允许容器保存中间数据的目录集合。当应用结束时这些数据会被清除。 |
| yarn.nodemanager.aux-services | 逗号分隔的服务名称 |  | 节点管理器运行的辅助服务集合。服务由定义在属性_yarn.nodemanager.auxservices.&lt;service-name&gt;.class_中的类实现。默认，没有指定的辅助服务。 |
| yarn.nodemanager.resource.memory-mb | int | 8192 | 节点管理器运行的容器被分配的物理内存（MB）的量。 |
| yarn.nodemanager.vmem-pmem-ratio | float | 2.1 | 容器虚拟内存和物理内存的比例。虚拟内存使用量可能超过这个分配量。 |
| yarn.nodemanager.resource.cpu-vcores | int | 8 | 节点管理运行的容器分配的CPU核心数量。 |

#### 3.2.8、Memory settings in YARN and MapReduce

相比于MapReduce 1使用的slot-based模型，YARN以更加细粒度（fine-grained）的方式对待内存。YARN不是一次指定运行在一个节点上固定最大数量的map和reduce slots，而是允许应用为某个task请求任意量的内存（在限制范围内）。在YARN模型中，节点管理器从一个pool分配内存，所以运行在某个节点上的tasks的数量依tasks的内存需要而定，而不是由固定的slots数量决定。

分配多少内存给节点管理器来运行容器由机器上的物理内存量决定。每个Hadoop daemon使用1000MB，所以对于一个datanode和节点管理器，总量是2000MB。为运行在机器上的其它进程保留足够的内存，并且通过设置配置属性_yarn.nodemanager.resource.memory-mb_（以MB为单位）为总分配量，余数可以分配给节点管理器的容器。（默认值8192MB，对大多数设置来说一般太小了。）

下一步是决定如何为单个jobs设置内存选项。有两种控制：一种是控制YARN分配给容器的内存；另一种是控制运行在容器里面的Java进程的堆内存大小。

注意：MapReduce内存都是由客户端在job中的配置控制的。YARN设置是集群级设置，并且不能被客户端改变。

容器内存大小由属性_mapreduce.map.memory.mb_和_mapreduce.reduce.memory.mb_决定，默认都是1024MB。这些设置都由application master在请求集群资源时使用，也由节点管理器使用，节点管理器运行和监控task容器。Java进程的对内存大小通过_mapred.child.java.opts_设置，默认值是200MB。也可以分别为map和reduce tasks设置Java选项（如**表10-4**）。

###### 表 10-4.MapReduce job memory properties\(set by the client\)

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| mapreduce.map.memory.mb | int | 1024 | map容器的内存量 |
| mapreduce.reduce.memory.mb | int | 1024 | reduce容器的内存量 |
| mapred.child.java.opts | String | -Xmx200m | 启动运行map和reduce tasks的容器使用的JVM选项。除了内存设置，例如，这个属性还可以包含用于故障排除的JVM属性。 |
| mapreduce.map.java.opts | String | -Xmx200m | 运行map tasks的子进程使用的JVM选项 |
| mapreduce.reduce.java.opts | String | -Xmx200m | 运行reduce tasks的子进程使用的JVM选项 |

例如，设置_mapred.child.java.opts_为**-Xmx800m**，而_mapreduce.map.memory.mb_还是默认值1024MB。当map task运行，节点管理器会分配一个1024MB容器（在task运行期间从pool大小中减少这个分配量），并且使用800MB最大堆内存的配置启动task JVM。注意，JVM进程会使用比堆内存大小大的内存量，并且这个开销由使用的native libraries，permanent generation space大小等等因素决定。重要的事情是JVM进程使用的物理内存（包括由它引起的任何进程，比如Streaming进程）不超过它的分配量（1024MB）。如果容器使用的内存超过了给它分配的量，它可能会被节点管理器终结并标记为失败。

YARN调度器使用一个最小或最大内存分配。默认最小1024MB（由_yarn.scheduler.minimum-allocation-mb_设置），默认最大8192MB（由_yarn.scheduler.maximum-allocation-mb_设置）。

也有容器必须满足的虚拟内存限制。如果容器的虚拟内存使用超过了分配的物理内存的指定倍数，节点管理器可能会终结这个进程。这个倍数由_yarn.nodemanager.vmem-pmem-ratio_属性设置，默认2.1。在前例中，可能到时task终结的虚拟内存上限为2150MB，即2.1×1024MB。

配置内存参数时，如果能在job运行期间监控task实际使用的内存将是非常有用的，通过MapReduce task counters可以实现。counters **PHYSICAL\_MEMORY\_BYTES**，**VIRTUAL\_MEMORY\_BYTES**，**COMMITTED\_HEAP\_BYTES**提供了内存使用的snapshot值，并且适合于在task attempt运行过程中进行观测。

Hadoop也提供控制MapReduce操作的内存的设置。在[混洗和排序]()章节已经提及。

#### 3.2.9、CPU settings in YARN and MapReduce

除了内存，YARN把CPU使用作为一个可管理的资源，应用可以请求它们需要的核心数量。节点管理器可以分配给容器的核心数量通过_yarn.nodemanager.resource.cpu-vcores_属性控制。应该被设置为机器上核心的总数减去机器上运行的daemon进程数（datanode、节点管理器、其它长期运行的进程，每个进程一个核心）。

MapReduce jobs可以通过_mapreduce.map.cpu.vcores_和_mapreduce.reduce.cpu.vcores_控制分配给map和reduce容器的核心分配。它们的默认值都是1，对于一般使用一个核心的单线程MapReduce tasks是合适的。

警告：尽管在调度过程中会追踪核心的数量（例如，因此一个容器不会被分配到一个没有空闲核心的机器上），默认情况下，节点管理器不会限制运行中的容器实际使用的CPU。这意味着，一个容器可能滥用超过分配指定量的CPU，可能导致同一主机值上的其它容器缺乏CPU。YARN支持使用Linux cgroups强制限制CPU使用。节点管理器的容器executor类（_yarn.nodemanager.container-executor.class_）必须被设置为使用**LinuxContainerExecutor**类，这个类必须被配置为使用cgroups（查看_yarn.nodemanager.linux-container-executor_中的属性）。

### 3.3、Hadoop Daemon Addresses and Ports

Hadoop daemons一般运行一个RPC服务（**表10-5**）用于daemons之间的通信，和一个HTTP服务（**表10-6**）用于为用户需求提供web页面。每个服务都由设置网络地址和监听端口号进行配置。端口号0指示服务在空闲端口上启动，但是通常不鼓励这样，因为这与设置集群防火墙策略不兼容。

###### 表 10-5.RPC server properties

| Property name | Default value | Description |
| --- | --- | --- |
| fs.defaultFS | file:/// | 当设置为HDFS URI时，这个属性决定namenode的RPC服务地址和端口。如果不指定端口，默认端口是8020。 |
| dfs.namenode.rpc-bind-host |  | namenode的RPC服务将会绑定到的地址。如果不设置（默认），绑定地址由_fs.defaultFS_决定。它可以设置为0.0.0.0让namenode监听所有的接口。 |
| dfs.datanode.ipc.address | 0.0.0.0:50020 | datanode的RPC服务地址和端口 |
| mapreduce.jobhistory.address | 0.0.0.0:10020 | job history服务的RPC服务地址和端口。这用于客户端（一般是集群外）来查询job history。 |
| mapreduce.jobhistory.bind-host |  | job history服务的RPC和HTTP服务会绑定到的地址 |
| yarn.resourcemanager.hostname | 0.0.0.0 | 资源管理器所运行在的机器的主机名。以下简称为${y.rm.hostname} |
| yarn.resourcemanager.bind-host |  | 资源管理器的RPC和HTTP服务会绑定到的地址。 |
| yarn.resourcemanager.address | ${y.rm.hostname}:8032 | 资源管理器的RPC服务地址和端口。由客户端（一般是集群外）使用用来和资源管理器通信。 |
| yarn.resourcemanager.admin.address | ${y.rm.hostname}:8033 | 资源管理器的admin RPC服务地址和端口。由admin client（通过 yarn rmadmin，一般在集群外运行）使用用来和资源管理器通信。 |
| yarn.resourcemanager.scheduler.address | ${y.rm.hostname}:8030 | 资源管理器调度器的RPC服务地址和端口。由application master用来和资源管理器进行通信。 |
| yarn.resourcemanager.resource-tracker.address | ${y.rm.hostname}:8031 | 资源管理器资源tracker的RPC服务地址和端口。由节点管理器用来和资源管理器进行通信。 |
| yarn.nodemanager.hostname | 0.0.0.0 | 节点管理器运行在的集群的主机名。以下简写为${y.nm.hostname}。 |
| yarn.nodemanager.bind-host |  | 节点管理器的RPC和HTTP服务会绑定到的地址。 |
| yarn.nodemanager.address | ${y.nm.hostname}:0 | 节点管理器的RPC服务地址和端口。由application master用来和节点管理器通信。 |
| yarn.nodemanager.localizer.address | ${y.nm.hostname}:8040 | 节点管理器的localizer的RPC服务地址和端口 |

###### 表 10-6.HTTP server properties

| Property name | Default value | Description |
| --- | --- | --- |
| dfs.namenode.http-address | 0.0.0.0:50070 | namenode的HTTP服务地址和端口 |
| dfs.namenode.http-bind-host |  | namenode的HTTP服务会绑定到的地址 |
| dfs.namenode.secondary.http-address | 0.0.0.0:50090 | 次要namenode的HTTP服务地址和端口 |
| dfs.datanode.http.address | 0.0.0.0:50075 | datanode的HTTP服务地址和端口（注意这个属性名与对应的namenode属性名不一致） |
| mapreduce.jobhistory.webapp.address | 0.0.0.0:19888 | MapReduce job history服务的地址和端口。这个属性在_mapred-site.xml_中设置 |
| mapreduce.shuffle.port | 13562 | 混洗handler的HTTP端口号。这用于提供map输出，并且不是一个用户可访问的web UI。这个属性在_mapred-site.xml_中设置 |
| yarn.resourcemanager.webapp.address | ${y.rm.hostname}:8088 | 资源管理器的HTTP服务地址和端口 |
| yarn.nodemanager.webapp.address | ${y.rm.hostname}:8042 | 节点管理器的HTTP服务地址和端口 |
| yarn.web-proxy.address |  | web app代理服务的HTTP服务地址和端口。如果不设置（默认），web app代理服务会在资源管理器中运行。 |

通常，设置服务的RPC和HTTP地址的属性由两种作用：决定服务绑定到的网络接口，并且被客户端或者集群中其它机器用来连接到这个服务。例如，节点管理器使用属性_yarn.resourcemanager.resource-tracker.address_来找到它们的资源管理器的地址。

经常会希望将服务绑定到多个网络接口，但是设置网络地址为0.0.0.0对服务是可行的，对于客户端或者集群中其它机器这个地址是无法解析的。一种解决方案是，分别为客户端和服务进行配置，但是更好的方式是为服务设置绑定主机。通过设置_yarn.resourcemanager.hostname_为主机名（外部可解析的）或者IP地址，并且_yarn.resourcemanager.bind-host_为0.0.0.0，可以确保资源管理器能绑定到机器上的任何地址，而同时为节点管理器和客户端提供一个可解析的地址。

处理RPC服务，datanodes运行一个TCP/IP服务用来进行block传输。服务地址和端口通过_dfs.datanode.address_属性设置，默认值是0.0.0.0:50010。

也有用于控制datanodes使用哪个网络接口作为它们的IP地址（HTTP和RPC服务的）。相关属性是_dfs.datanode.dns.interface_，默认使用的是默认的网络接口。可以将这个属性明确的设置为某个特定接口地址（例如，_eth0_）。

### 3.4、Other Hadoop Properties

#### 3.4.1、Cluster membership

为额外辅助和未来移除节点，可以指定一个包含授权的机器（可以作为datanodes或者节点管理器加入集群）的文件。使用_dfs.hosts_和_yarn.resourcemanager.nodes.include-path_（分别为datanodes和节点管理器）属性指定这个文件，并且对应的_dfs.hosts.exclude_和_yarn.resoucemanager.nodes.exclude-path_属性指定包含正式停止使用的节点的文件。

#### 3.4.2、Buffer size

Hadoop为它的I/O操作使用4KB（4096字节）的缓存。这是一个保守的设置，使用现代化的硬件和操作系统，增加这个值通常能够提升性能；128KB是一个常用的选择。在_core-site.xml_文件中使用_io.file.buffer.size_属性以字节为单位设置它。

#### 3.4.3、HDFS block size

HDFS block默认大小128MB，但是许多集群使用更大的block大小（例如，256MB）来减轻namenode的内存压力并且让mappers处理更多的数据。使用_hdfs-site.xml_中的_dfs.blocksize_属性指定。

#### 3.4.4、Reserved storage space

默认情况下，datanodes会尽量使用它们的存储目录中的所有可用空间。如果要保留一些存储空间给非HDFS使用，可以设置属性_dfs.datanode.du.reserved_为以字节为单位的要保留的空间量。

#### 3.4.5、Trash

Hadoop文件系统有一个垃圾处理功能（trash facility），Hadoop删除的文件没有实际删除，而是移动到垃圾文件夹，在被系统永久删除之前会在那里保留一段最小时间。最小时间由_core-site.xml_中的_fs.trash.interval_以分钟为单位设置。默认0，即不启用垃圾文件夹。

与许多操作系统类似，Hadoop的垃圾处理功能是一个用户级别的特征，意味着只有使用文件系统shell删除的文件会放进垃圾文件夹。程序删除的文件则直接删除。但是，通过构建**Trash**实例，可以在程序中使用trash功能，把要删除的文件**Path**作为参数调用实例的_moveToTrash\(\)_方法。方法会返回一个代表成功的值，返回_false_意味着没有启用trash功能或者文件已经在trash文件夹中。

当启用了trash，每个用户都有自己的trash目录，叫做_.Trash_，位于home目录中。文件恢复很简单：从_.Trash_的子目录找到指定文件，并将它从trash子树（subtree）移出。

HDFS会自动删除trash文件夹中的文件，但是其它文件系统不会，所以，必须设置定期保证文件删除。可以_expunge_ trash，这个操作会删除trash文件夹中超过最小时间的文件，使用文件系统shell：

```shell
% hadoop fs -expunge
```

**Trash**类的_expunge\(\)_方法也有同样的效果。

#### 3.4.6、Job scheduler

尤其是在多用户的集群设置中，要根据组织的需求更新job调度队列的配置。例如，可以为使用集群的每个group设置一个队列。详见[YARN调度章节]()。

#### 3.4.7、Reduce slow start

默认情况下，当job中5%的map tasks完成后，调度器才会为这个job调度reduce tasks。对于大型的jobs，这会导致集群使用率的问题，因为在等待map tasks完成时，map tasks会占用reduce的容器。设置属性*mapreduce.job.reduce.slowstart.completedmaps*为比较高的值，比如0.80（80%），能提供吞吐量。

#### 3.4.8、Short-circuit local reads

当从HDFS读取一个文件时，客户端联系datanode并且数据通过TCP连接发送到客户端。如果要读取的block位于客户端所在的节点上，对于客户端来说绕过网络直接从硬盘读取block的数据更加高效。这叫做*short-circuit local read*，本地短路读取，可以使像HBase这样的应用性能更好。

通过设置属性*dfs.client.read.shortcircuit*为**true**来开启本地短路读取。本地短路读取通过使用Unix domain sockets实现，它使用本地路径来进行client-datanode通信。这个路径通过属性*dfs.domain.socket.path*设置，并且必须是一个只有datanode用户（一般是hdfs）或者*root*用户可以创建的目录，例如*/var/run/hadoop-hdfs/dn_socket*。

