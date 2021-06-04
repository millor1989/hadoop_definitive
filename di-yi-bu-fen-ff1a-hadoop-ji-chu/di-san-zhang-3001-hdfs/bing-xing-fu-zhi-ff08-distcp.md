## 7、使用distcp并行复制（Parallel Copying with distcp）

截至目前介绍的HDFS访问模式聚焦在单线程访问。尽管通过使用通配符可以访问一组文件，但是，如果要高效并行的处理这些文件，将不得不自己动手写程序。Hadoop有一个有用的程序叫做***distcp***可以并行的复制数据进出（to and from）Hadoop文件系统。

**用法：**

```shell
hadoop distcp OPTIONS [source_path...] <target_path>
```

***distcp***可作为***hadoop fs -cp***命令的一个高效替代，可以用于**复制文件**：

```shell
hadoop distcp file1 file2
```

也可以用于**复制目录**：

```shell
hadoop distcp dir1 dir2	
```

如果目录*dir2*不存在，会创建它，并把*dir1*的内容复制到里面。可以指定多个源路径，所有内容都会复制到目标路径。

如果已经存在*dir2*，会被*dir1*复制到*dir2*中，即创建目录结构*dir2/dir1*。可以使用***-overwrite***选项强制覆盖；也可使用***-update***选项复制目的目录不存在的文件和目录。例如：

```shell
hadoop distcp -update dir1 dir2
```

***distcp***的实现是MapReduce job，通过在集群中并行运行的maps程序实现复制工作。没有reducers。每个文件使用一个map，***distcp***尝试给每个map分配近似相同的数据量。默认情况下，使用多至20个maps，但是可以使用***-m***参数改变maps最大数量。

***distcp***非常常用的一个案例是在两个HDFS集群之间传输数据。例如：

```shell
hadoop distcp -update -delete -p hdfs://namenode1/foo hdfs://namenode2/foo
```

***-delete***选项会使***distcp***删除目的目录中存在源目录不存在的所有文件。***-p***的含义是保留文件的状态属性（例如，权限、block大小、复本）。可以执行没有参数的***distcp***命令来查看使用说明。

如果两个集群运行不同版本的HDFS，可以用***webhdfs***协议执行distcp操作：

```shell
hadoop distcp webhdfs://namenode1:50070/foo webhdfs://namenode2:50070/foo
```

另一个变种是使用HttpFs代理作为***distcp***源或者目的（再使用webhdfs 协议），这样就有设置防火墙和带宽控制的优势。

### 7.1、保持HDFS集群平衡（Keeping an HDFS Cluster Balanced）

往HDFS复制数据的时候，要考虑到集群的平衡。当文件的blocks在集群中均匀分布是HDFS工作状态最佳，要确保***distcp***不破坏这种平衡。例如，当指定了 ***-m 1***，会使用一个map执行复制，这时，除了慢且不能高效利用集群资源外，也意味着每个block的第一个复本会放置在运行map的节点上（直到放满磁盘），第二、第三复本会在整个集群分布，但是这个节点会不平衡；使用超过集群节点的maps数量可以避免这种情况的出现。因为这个原因，最好使用默认的（每个节点？？）20个maps来启动***distcp***。

但是，可能不能总是防止集群变得不平衡。也许会为了某些节点可以执行其它的jobs而限制maps的数量。这时，可以使用***balancer***工具随后查看集群的block分布情况。

