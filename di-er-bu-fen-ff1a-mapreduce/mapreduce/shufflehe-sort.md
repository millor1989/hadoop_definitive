## 3、混洗和排序（Shuffle and Sort）

MapReduce 保证每个 reducer 的输入是按照 key 进行排序的。系统执行排序——和传输 map 输出到 reducer 作为输入——的过程称为混洗（shuffle）。

### 3.1、Map侧（The Map Side）

当 map 函数开始产生输出时，它不是简单的写到磁盘，这个过程还包括并且利用内存中的缓冲写（buffering write）的优势和为了高效做一些预排序。如图：

###### 图 7-4 Shuffle and sort in MapReduce

![](/assets/1571480856962.png)

每个 map task 都有一个循环内存缓存（circular memory buffer）用来写输出。这个缓存默认 100M（通过_mapreduce.task.io.sort.mb_属性设置）。当这个缓存的内容达到一定的阈值（_mapreduce.map.sort.spill.percent_，默认0.8或80%），一个后台线程会开始将内容溢出（_spill_）到磁盘。溢出发生时，map 输出会继续写到这个缓存，但是如果缓存被填满，map 会阻塞直到溢出完成。溢出内容被以循环的风格写到 _mapreduce.cluster.local.dir_ 属性指定的目录中的一个 job 专属的子目录。

在写到磁盘之前，线程首先根据数据最终会被送到的 reducers 的数量将数据分区。每个分区内，后台进程执行一个在内存中的按 key 排序，如果存在 combiner 函数，那么它是在排序的输出上运行的。运行 combiner 函数会使 map 输出更紧凑，那么就有更少的数据写到本地磁盘和传到 reducer。

每当内存缓存达到溢出阈值，创建一个新的溢出文件，所以在 map task 写完它的最后一条输出记录，可能会有几个溢出文件。在 map task 结束前，溢出文件会被合并到一个分区的并且是排序的输出文件。配置属性_mapreduce.task.io.sort.factor_ 控制一次合并的 streams（流？？）的数量；默认值 10。

如果有至少 3 个溢出文件（_mapreduce.map.combine.minspills_ 属性设置），combiner 会在写输出文件之前再次运行。再次调用 combiners 可能会在输入上重复运行而不影响最终结果。如果只有一个或两个溢出文件，map 输出大小的减少预期将不值得调用 combiner 的开销，所以不会为这个 map output 再次运行。

写 map 输出到磁盘上时最好进行压缩，这样可以写得更快、节省磁盘空间、减少传输到 reducer 的数据量。默认 map 输出是不压缩的，设置 _mapreduce.map.output.compress_ 为**true**开启。通过 _mapreduce.map.output.compress.codec_ 属性来设置压缩库。

输出文件的分区通过 HTTP 对 reducers 变得可用。服务文件分区的最大 worker 线程由 _mapreduce.shuffle.max.threads_ 属性控制；这个设置是为每个节点管理器的，而不是为每个 map task 的。默认值 0 设置最大线程数为机器上处理器数量的两倍。

### 3.2、Reduce侧（The Reduce Side）

map 输出文件保存在运行 map task 机器的本地磁盘上（尽管 map 输出写在本地磁盘上，reduce 输出可能不是），但是在 reduce 阶段需要为分区运行 reduce task。此外，reudce task 需要集群中几个 map tasks 的输出的特定分区。map tasks 可能在不同时间完成。reduce task 在每个 map task 完成时开始复制的它们的输出。这是 reduce task 的 _**copy phase**_（复制阶段）。reduce task 有少数几个（默认个数 5，通过 *mapreduce.reduce.shuffle.parallelcopies* 属性设置）复制线程（copier threads）可以并行获取 map 输出。

reducers 是怎么知道从哪个机器获取 map 输出的呢？当 map tasks 成功完成，它们用心跳机制通知 application master。因而，对于某个 job，application master 知道 map 输出和主机（hosts）之间的映射。reducer 中的一个进程定期向 application master 请求输出主机直到获取所有的输出。当第一个 reducer 获取 map 输出后主机不会删除 map 输出，因为 reducer 可能会失败。只有 application master 通知主机删除 map 输出文件时，才会删除，这发生在 job 完成后。

如果 map 输出足够小（缓存大小由 _mapreduce.reduce.shuffle.input.buffer.percent_ 属性控制，指定了为这一目的使用的堆的比例）就直接复制到 reduce task 的 JVM 内存中，否则就将 map 输出复制到磁盘。当内存中缓存达到一定阈值（由 _mapreduce.reduce.shuffle.merge.percent_ 属性控制）或达到 map 输出的一个阈值（_mapreduce.reduce.merge.inmem.threshold_），在 merge 中会运行这个缓存以减少写到磁盘的数据量。

随着复制文件在磁盘上积累，一个后台线程把它们合并到更大的、排序的（多个）文件中。这节省一些后续合并的时间。注意，为了进行 map 输出合并，任何压缩的 map 输出都必须在在内存中进行解压。

当所有的 map 输出都复制完，reduce task 进入 _**sort phase**_（_排序阶段_，叫做 _**merge phase**_ 合并阶段更合适，因为排序是在 map 侧进行的），在这个阶段合并 map 输出，维护它们的排序顺序。这个是循环进行的。例如，如果有 50 个map输出并且 _merge factor_ 是10（_mapreduce.task.io.sort.factor_），会进行 5 轮合并。每一轮合并 10 个文件为一个文件，最终会有 5 个中间文件。最后，不会将 5 个中间文件合并为一个文件并写到磁盘，而是将合并结果传递给最后的阶段：_**reduce pahse**_。最后的合并可以是内存中数据和磁盘上数据的混合。

每一轮合并文件的数量事实上比这个例子要灵巧。目的是合并最少的文件，以在最终一轮合并中合并 merge factor 个文件。所以，如果有 40 个文件，合并将不是每次合并 10 个文件以获取 4 个中间文件。而是，第一轮只合并 4 个文件，后续三轮合并 10 个文件，4 个中间文件和 6 个没有进行合并的文件（共10个文件）进行最后一轮合并。注意，这并没有改变进行合并的轮数，只是一个获取写到磁盘数据量最小的优化，因为最终一轮总是直接合并结果到 reduce 阶段。

在 reduce 阶段，为排序的输出中的每个 key 调用 reduce 函数。reduce 阶段的输出直接写到输出文件系统，通常是 HDFS。在使用 HDFS 的场合，因为节点管理器也运行一个 datanode，第一个 block replica 会写到本地磁盘。

### 3.3、配置调试（Configuration Tuning）

###### 表 7-1 Map-side tuning properties

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| mapreduce.task<br/>.io.sort.mb | int | 100 | 排序map输出时使用的内存缓存（MB） |
| mapreduce.map<br/>.sort.spill.percent | float | 0.80 | 触发内存缓存溢出到磁盘的阈值（map输出使用的内存缓存的比例） |
| mapreduce.task<br/>.io.sort.factor | int | 10 | 排序文件时一次合并的流的最大数目。这个属性在reduce中也可以使用。把它设置为100相当常见。 |
| mapreduce.map<br/>.combine.minspills | int | 3 | （如果指定了combiner）触发combiner运行的最小溢出文件数目 |
| mapreduce.map<br/>.output.compress | boolean | false | 是否压缩map输出 |
| mapreduce.map<br/>.output.compress.codec | _**class name**_ | org.apache.hadoop.io<br/>.compress.DefaultCodec | 用于map输出压缩的压缩codec |
| mapreduce.shuffle<br/>.max.threads | int | 0 | 每个节点管理器中提供map输出给reducers的工作者线程个数。这个属性是集群的，不能在单独job中设置。0意味着使用Netty默认的可用处理器的个数的两倍个数的线程。 |

###### 表 7-1 Reduce-side tuning properties

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| mapreduce.reduce.shuffle<br/>.parallelcopies | int | 5 | 复制map输出到reducer的线程数量 |
| mapreduce.reduce.shuffle<br/>.maxfetchfailures | int | 10 | reducer尝试获取map输出的允许的失败次数 |
| mapreduce.task.io.sort.factor | int | 10 | 排序文件时一次合并流的最大数量。这个属性在map中也可用 |
| mapreduce.reduce.shuffle<br/>.input.buffer.percent | float | 0.70 | 在混洗的复制阶段给map输出缓存分配的内存占整个堆内存的比例 |
| mapreduce.reduce.shuffle<br/>.merge.percent | float | 0.66 | 内存中的map输出缓存达到这个阈值时，启动合并map输出并溢出到磁盘的处理。这个阈值是内存中的map输出占mapreduce.reduce.input.buffer.percent属性所定义的内存大小的比例。 |
| mapreduce.reduce.merge<br/>.inmem.threshold | int | 1000 |  |
| mapreduce.reduce.input<br/>.buffer.percent | float | 0.0 | 在reduce阶段中用于保存map输出的内存占整个堆内存的比例。为了启动reduce阶段，内存中map输出的大小必须小于这个百分比。默认情况下，所有的map输出在reduce开始前都合并到磁盘，以给reducer尽可能多的内存。可是，如果reducer需要较少的内存，可以增加这个百分比设置，以减少合并到磁盘的map输出的量。 |

一般原则是给混洗尽可能多的内存。可是，有一个权衡，要确保 map 和 reduce 函数有足够的内存运行。这就是最好使 map 和 reduce 函数使用尽可能少内存的原因——当然它们不能使用无限定量的内存（例如避免在 map 中使用累积值）。

给予运行 map 和 reduce task 的 JVM 的内存的量通过 mapred.child.java.opts 属性设置。应该确保这个值尽可能和 task 节点上的内存一样大。

在 map 侧，通过避免多重溢出到磁盘（一个最佳）可以获得最佳的性能。如果可以估计 map 输出的大小，可以设置 _mapreduce.task.io.sort.\*\*_ 属性来使溢出最小化。特别是，如果可以，应该增加 _mapreduce.task.io.sort.mb_。有一个 MapReduce counter（SPILLED\_RECORDS）记录在 job 过程中溢出到磁盘的记录总数。注意，这个 counter 包含了 map 侧和 reduce 侧的溢出。

在 reduce 侧，当中间数据全部都在内存中时能够获得最佳性能。默认情况下不会发生这种情况，因为常见的情形是所有的内存都保留以用于 reduce 函数。但是如果 reduce 函数内存需求很少，设置_mapreduce.reduce.merge.inmem.threshold_ 为 0 并且 _mapreduce.reduce.input.buffer.percent_ 为 1.0（或者一个稍小的值）可能会带来性能的爆发。

一般地，Hadoop 默认使用 4KB 的缓存，很少，应该增加集群中的这个值（通过设置 _io.file.buffer.size_）。

