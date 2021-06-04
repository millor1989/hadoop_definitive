## 4、Task执行（Task Execution）

### 4.1、Task执行环境（The Task Execution Environment）

Hadoop提供map或reduce task运行的环境信息。例如，map task可以获取它正在处理的文件名字，map或reduce task能够获取task的attempt数量。**表7-3**中的属性可以从job的配置中访问，在旧的MapReduce API中通过提供一个**Mapper**或**Reducer**的*configure()*方法的实现获取。在新的API中，这些属性可以通过传递给**Mapper**或**Reducer**的所有方法的*context*对象访问。

###### 表 7-3 Task environment properties

| Property name             | Type    | Description        | Example                              |
| ------------------------- | ------- | ------------------ | ------------------------------------ |
| mapreduce.job.id          | String  | job ID             | job_200811201130_0004                |
| mapreduce.task.id         | String  | task ID            | task_200811201130_0004_m_000003      |
| mapreduce.task.attempt.id | String  | task attempt ID    | attempt_200811201130_0004_m_000003_0 |
| mapreduce.task.partition  | int     | task在job中的索引  | 3                                    |
| mapreduce.task.ismap      | boolean | task是否是map task | true                                 |

#### 4.1.1、流环境变量（Streaming environment variables）

Hadoop设置job配置参数为流程序的环境变量。可是，它用下划线替换非字母数字的字符来获得合法的环境变量名。如下Python表达式表示了如何获取一个Python流的*mapreduce.job.id*属性的值：

```python
os.environ["mapreduce_job_id"]
```

也可以通过启动MapReduce时补充***-cmdenv***选项到流启动程序来为流处理设置环境变量（一次设置一个环境变量）。例如，如下代码设置**MAGIC_PARAMETER**环境变量：

```shell
-cmdenv	MAGIC_PARAMETER=abracadabra
```

### 4.2、推测执行（Speculative Execution）

MapReduce模型是把job拆分为tasks并且并行执行这些tasks以使job整体执行时间少于串行运行tasks时的时间。这使job执行时间队运行慢的tasks敏感，因为一个运行慢的task可能会使整个job执行时间显著地变长。当job由成百上千的tasks组成时，出现掉队tasks的可能性非常大。

Tasks慢的原因多种多样，包括硬件陈旧或者软件配置错误，但是因为task仍然成功完成所以原因很难检测到，尽管比期望的时间长。Hadoop不会尝试诊断并且修复运行慢的tasks；而是，它尝试在检测到比预期运行慢的task时启动一个等价的task作为备份。这叫做tasks的推测执行（***speculative execution***）。

推测执行不是同时启动两个重复的tasks并比赛哪个先完成。而是，调度器跟踪job中同类型（map or reduce）所有tasks的进度，只为比平均运行时间明显要长的一小部分tasks启动推测备份执行。当一个task成功完成，运行中的重复task因为不再被需要会被杀死。所以，如果原始task先完成，杀死推测执行的task，反之亦然。

推测执行只是一个优化，不是让jobs更可靠运行的特性。如果有些bugs有时会导致task挂起或运行时间长，依靠推测执行来避免问题是不明智的，因为推测执行中bugs仍然存在并可能影响执行，此时应该修复bugs。

推测执行默认是开启的。可以在集群层面独立的为map或reduce task分别开启或关闭。相关属性如**表7-4**:

###### 表7-4 Speculative execution properties

| Property name                                  | Type    | Default value                                                |
| ---------------------------------------------- | ------- | ------------------------------------------------------------ |
| mapreduce.map.speculative                      | boolean | true                                                         |
| mapreduce.reduce.speculative                   | boolean | true                                                         |
| yarn.app.mapreduce.am.job.speculator.class     | class   | org.apache.hadoop.mapreduce.v2.app.speculate.DefaultSpeculator |
| yarn.app.mapreduce.am.job.task.estimator.class | class   | org.apache.hadoop.mapreduce.v2.app.speculate.LegacyTaskRuntimeEstimator |

什么情况下会希望关闭推测执行呢？推测执行的目标是减少job执行时间，但是这事宜集群效率为代价的。在繁忙的集群上，推测执行会减少集群整体的吞吐量，因为冗余的tasks的执行仅仅减少了一个job的执行时间。因为这个原因，某些集群管理员更愿意关闭集群的推测执行并让用户明确地为特定的jobs开启推测执行。这对老版本的Hadoop尤其相关，老版本Hadoop在调度推测执行任务时会过于有侵占性。

为reduce tasks关闭推测执行的好的原因是，任何重复的reduce tasks都必须像原始task那样获取相同的map输出，这会显著地增加集群的网络交通。

关闭推测执行的另一个原因是为了非等幂（nonidempotent）tasks。可是，很多情况下可能会写等幂的tasks并且在task成功时使用**OutputCommitter**来推送输出到它的最终位置。

### 4.3、Output Committers

Hadoop MapReduce使用一个commit协议来清晰地确保jobs和tasks成功或失败。这个行为通过使用**OutputCommitter**为job实现，在旧的MapReduce API中通过调用**JobConf**的*setOutputCommitter()*方法或者配置属性***mapred.output.committer.class***来设置。在新的MapReduce API中，**OutputCommitter**由**OutputFormat**通过它的*getOutputCommitter()*方法决定。默认值是**FileOutputCommitter**，适用于基于文件的MapReduce。如果需要为jobs或tasks进行特别的设置或清理，可以个性化一个已经存在的OutputCommitter或者写一个新的实现。

OutputCommitter API（新旧MapReduce API都一样）如下：

```java
public abstract class OutputCommitter {
  public abstract void setupJob(JobContext jobContext) throws IOException;
  public void commitJob(JobContext jobContext) throws IOException { }
  public void abortJob(JobContext jobContext, JobStatus.State state)
      throws IOException { }
  public abstract void setupTask(TaskAttemptContext taskContext)
      throws IOException;
  public abstract boolean needsTaskCommit(TaskAttemptContext taskContext)
      throws IOException;
  public abstract void commitTask(TaskAttemptContext taskContext)
      throws IOException;
  public abstract void abortTask(TaskAttemptContext taskContext)
      throws IOException;
  }
}
```

*setupJob()*方法在job开始运行前调用，一般用于进行初始化。对于**FileOutputCommitter**，这个方法创建了最终输出目录*${mapreduce.output.fileoutputformat.outputdir}*和一个子目录*_temporary*作为用于task输出的临时工作空间。

如果job成功，调用*commitJob()*方法，在默认的基于文件的实现中会删除临时工作空间，并且在输出目录中创建一个叫作*_SUCCESS*的隐藏的空的标记文件，用来指示文件系统客户端job成功完成。如果job不成功，会使用一个指示job失败或被杀死的状态对象（**JobStatus.State**）调用*abortJob()*方法。在默认实现中，这也会删除job的临时工作空间。

task级别的操作也是类似的。在task运行前调用*setupTask()*方法，默认实现没有作任何操作，因为task输出的临时目录在写task输出时才创建。

tasks的commit阶段是可选的，并且通过从*needsTaskCommit()*方法返回false可以关闭这个阶段。这样，会省去框架为task执行分布式commit协议（distributed commit protocol），并且*commitTask()*和*abortTask()*方法都不会被调用。当task没有写输出时**FileOutputCommitter**会跳过commit阶段。

如果task成功，调用commitTask()方法，默认实现是将临时task输出目录（目录名包含task attempt ID以避免task attempt之间冲突）移动到最终输出路径*${mapreduce.output.fileoutputformat.outputdir}*。如果task失败，框架调用*abortTask()*方法，删除task临时输出目录。

对于某个task如果有多个task attempts，框架确保只有一个会committed，其它的会被抛弃。这种情形在第一个attempt因为某种原因失败时会发生——这时，它会被抛弃，稍后的成功的attempt会被committed。也有两个tasks作为推测执行tasks同时运行的情况，此时，首先完成的task会被committed，另一个会被抛弃。

#### 4.3.1、Task副产品文件（Task side-effect files）

从map和reduce tasks写输出的常用方法是使用**OutputCollector**收集键值对。某些应用需要比键值对模型更多的灵活性，所以这些应用从map或reduce task直接把输出文件写到分布式文件系统，比如HDFS。

需要确保同一个task的多个实例不写同一个文件。如上节内容所述，**OutputCommitter**协议解决了这个问题。如果应用在它们的tasks的工作目录写了副产品文件，成功完成的tasks的副产品文件会被自动推送到输出目录，而失败tasks的副产品文件会被删除。

通过job配置的*mapreduce.task.output.dir*属性可以获取task的工作目录。或者，MapReduce程序可以使用Java API——调用**FileOutputFormat**的静态方法*getWorkOutputPath()*来获取代表工作目录的**Path**对象。框架会在执行task之前自动创建工作目录，所以不需要手动创建。

