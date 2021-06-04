## 1.Counters

对于所分析的数据，总会有想要了解的，相对于所执行分析来说是次要的某些信息。例如，计数无效记录并发现无效记录在整个数据集中所占的比例非常高——或许检测记录无效的那部分程序有bug？或者，如果数据质量非常低并且真的有许多无效记录，在发现这种情况后，可能会需要决定增加数据集的大小，以便有效记录的数量对于产生有意义的分析来说足够大。

counters是一个收集job数据（关于质量控制或关于应用级别的统计数据）的有用渠道。对问题诊断也很有用。如果想要在map或reduce task中输出一条日志信息，最好首先考虑是否可以用counter来替代用以记录某个特定状况的发生。除了对于大数据集的jobs counter的值比log输出的值更容易获取外，还能获取对应状况发生的次数，而要从日志文件中获取某个状况的次数需要做许多工作。

### 1.1、内置Counters（Built-in Counters）

Hadoop为每个job维护了一些内置的counters，这些counters报告多种矩阵（metrics）。例如，有些counters关于处理的记录的条数和字节数量，可以通过它确认输入的消费量和输出的产生量。

counters被分为组，内置counters的几个组如**表9-1**:

###### 表 9-1.Built-in counter groups

| Group                     | Name/Enum                                                    | Reference |
| ------------------------- | ------------------------------------------------------------ | --------- |
| MapReduce task counters   | org.apache.hadoop.mapreduce.TaskCounter                      | 表 9-2    |
| Filesystem counters       | org.apache.hadoop.mapreduce.FileSystemCounter                | 表 9-3    |
| FileInputFormat counters  | org.apache.hadoop.mapreduce.lib.input.FileInputFormatCounter | 表 9-4    |
| FileOutputFormat counters | org.apache.hadoop.mapreduce.lib.output.FileOutputFormatCounter | 表 9-5    |
| Job counters              | org.apache.hadoop.mapreduce.JobCounter                       | 表 9-6    |

每个组要么包含*task counters*（作为task进度更新）要么包含*job counters*（作为job进度更新）。

#### 1.1.1、Task counters

task counters收集task执行过程中的信息，结果是聚合一个job中的所有tasks信息。例如，**MAP_INPUT_RECORDS** counter，它计数每个map task读取的记录，并且聚合一个job中所有的map tasks，所以最终数字是整个job输入记录的总数。

task counters由每个task attempt维护，并且定期地向application master汇报以便进行全局聚合。task counters每次都是整体地发送，而不是发送上次改变后的计数，因为这样能够避免丢失信息引起的错误。另外，在job运行过程中，如果task失败，counters可能会变小。

counter的值只在job成功完成后一次性地确定。但是，某些counters在task运行中提供有用的诊断信息，可以通过web UI监控它们。例如，**PHYSICAL_MEMORY_BYTES**，**VIRTUAL_MEMORY_BYTES**，和**COMMITTED_HEAP_BYTES**指示某个task attempt运行过程中的内存使用变化。

###### 表 9-2.Built-in MapReduce task counters

| Counter                                          | Description                                                  |
| ------------------------------------------------ | ------------------------------------------------------------ |
| Map输入记录（MAP_INPUT_RECORDS）                 | job中所有maps消费的输入记录数。每当从**RecordReader**读取一条记录并传递记录到map的*map()*方法时增加。 |
| 分片原字节（SPLIT_RAW_BYTES）                    | maps读取的输入分片对象的字节数。这些对象代表分片元数据（即，一个文件中的偏移量和长度）而不是分片数据本身，所有，总的大小应该很小。 |
| Map输出记录（MAP_OUTPUT_RECORDS）                | job中所有的maps产生的map输出记录数。每当调用map的**OutputCollector**的*collect()*方法时增加。 |
| Map输出字节（MAP_OUTPUT_BYTES）                  | job中所有maps产生的未压缩输出的字节数。每当调用map的**OutputCollector**的*collect()*方法时增加。 |
| Map输出实际字节（MAP_OUTPUT_MATERIALIZED_BYTES） | map输出实际写到磁盘上的字节数。如果开启了map输出压缩，会反映在counter的值上。 |
| Combine输入记录（COMBINE_INPUT_RECORDS）         | job中所有combiners（如果有）消费的数据记录数。每当从combiner的迭代器读取值时增加。注意，这个值是combiner消费的值的数量，不是去重键分组的数量（它不是有用的metric，因为没有必要每个键分组一个combiner）。 |
| Combine输出记录（COMBINE_OUTPUT_RECORDS）        | job中所有combiners（如果有）产生的输出记录数。每当调用combiner的**OutputCollector**的*collect()*方法时增加。 |
| Reduce输入组（REDUCE_INPUT_GROUPS）              | job中所有reducer消费的去重键组数。每当调用reducer的*reduce()*方法时增加。 |
| Reduce输入记录（REDUCE_INPUT_RECORDS）           | job中所有reducers消费的输入记录数。每当从reducer的迭代器读取值时增加。如果reducers消费了所有的输入，这个数应该与map输出记录数相同。 |
| Reduce输出记录（REDUCE_OUTPUT_RECORDS）          | job中所有reducers产生的reduce输出记录数。每当调用reducer的**OutputCollector**的*collect()*方法时增加。 |
| Reduce混洗字节（REDUCE_SHUFFLE_BYTES）           | 通过混洗从map输出复制到reducers的字节数                      |
| Spilled记录（SPILLED_RECORDS）                   | job中所有的map和reduce tasks溢出到磁盘的记录数               |
| CPU毫秒(CPU_MILLISECONDES)                       | 以毫秒为单位的task累积CPU时间，与*/proc/cpuinfo*中的相同。   |
| 物理内存字节（PHYSICAL_MEMORY_BYTES）            | 以字节为单位的task使用的物理内存，与*/proc/meminfo*中的相同。 |
| 虚拟内存字节（VIRTUAL_MEMORY_BYTES）             | 以字节为单位的task使用的虚拟内存，与*/proc/meminfo*中的相同。 |
| Committed堆字节（COMMITTED_HEAP_BYTES）          | 以字节为单位的JVM中所有可用内存，与*Runtime.getRuntime().totalMemory()*获取的一致。 |
| GC时间毫秒（GC_TIME_MILLIS）                     | 以毫秒为单位的tasks中的垃圾回收耗费时间，与*GarbageCollectionMXBean.getCollectionTime()*获取的一致。 |
| 混洗maps（SHUFFLED_MAPS）                        | 通过混洗传送到reducers的map输出文件数。                      |
| 失败的混洗（FAILED_SHUFFLE)                      | 混洗中map输出复制失败数。                                    |
| 合并的map输出（MERGED_MAP_OUTPUTS）              | 混洗的reduce侧合并的map输出数。                              |

###### 表 9-3.Built-in filesystem task counters

| Counter                              | Description                                                  |
| ------------------------------------ | ------------------------------------------------------------ |
| 读取文件系统字节（BYTES_READ）       | map和reduce tasks通过文件系统读取的字节数。每个文件系统都有一个counter，文件系统可以是Local、HDFS、S3、等。 |
| 写文件系统字节（BYTES_WRITTEN）      | map和reduce tasks通过文件系统写的字节数。                    |
| 文件系统读取操作（READ_OPS）         | map和reduce tasks通过文件系统进行的读操作（例如，打开、读取文件状态）数。 |
| 文件系统大的读操作（LARGE_READ_OPS） | map和reduce tasks通过文件系统进行的大的读操作（例如，列出某个大目录的目录）数。 |
| 文件系统写操作（WRITE_OPS）          | map和reduce tasks通过文件系统进行的写操作（例如，创建、追加）数。 |

###### 表 9-4.Built-in FileInputFormat task counters
| Counter                | Description                                    |
| ---------------------- | ---------------------------------------------- |
| 读取字节（BYTES_READ） | map tasks通过**FileInputFormat**读取的字节数。 |

###### 表 9-5.Built-in FileOutputFormat task counters
| Counter                 | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| 写字节（BYTES_WRITTEN） | map tasks（只有map的jobs）或者reduce tasks通过FileOutputFormat写的字节数。 |

#### 1.1.2、Job counters

job counters是由application master维护的，所以它们不需要通过网络传送，不同于其它counters（包括用户定义的counters）。它们统计job级别的数据，不是随着task运行变化的值。例如，**TOTAL_LAUNCHED_MAPS**计数job运行过程中启动的map tasks（包括失败的tasks）。

###### 表 9-6. Built-in job counters

| Counter                                      | Description                                                  |
| -------------------------------------------- | ------------------------------------------------------------ |
| 启动的map tasks（TOTAL_LAUNCHED_MAPS）       | 启动的map tasks数（包括启动的推测执行tasks和失败的tasks）。  |
| 启动的reduce tasks（TOTAL_LAUNCHED_REDUCES） | 启动的reduce tasks数（包括启动的推测执行tasks和失败的tasks）。 |
| 启动的uber tasks（TOTAL_LAUNCHED_UBERTASKS） | 启动的uber tasks数。                                         |
| uber tasks中的maps（NUM_UBER_SUBMAPS）       | uber tasks中的maps数。                                       |
| uber tasks中的reduces（NUM_UBER_SUBREDUCES） | uber tasks中的reduces数。                                    |
| 失败的map tasks（NUM_FAILED_MAPS）           | 失败的map tasks数。                                          |
| 失败的reduce tasks（NUM_FAILED_REDUCES）     | 失败的reduce tasks数。                                       |
| 失败的uber tasks（NUM_FAILED_UBERTASKS）     | 失败的uber tasks数。                                         |
| 杀死的map tasks（NUM_KILLED_MAPS）           | 被杀死的map tasks数。                                        |
| 杀死的reduce tasks（NUM_KILLED_REDUCES）     | 被杀死的reduce tasks数。                                     |
| 数据本地的map tasks（DATA_LOCAL_MAPS）       | 与输入数据在同一个节点上运行的map tasks数。                  |
| 机架本地的map tasks（RACK_LOCAL_MAPS）       | 与输入数据在同一个机架的不同节点上运行的map tasks数。        |
| 其它本地性map tasks（OTHER_LOCAL_MAPS）      | 与输入数据在不同机架的节点上运行的map tasks数。跨机架的带宽是稀缺资源，Hadoop尝试将map tasks放在离它们输入数据近的节点，这个counter的值应该尽量低。 |
| map tasks总时间（MILLIS_MAPS）               | 以毫秒为单位的map tasks运行总时间。包括推测执行的tasks。有对应的统计核心和内存使用时间的counters（VCORES_MILLIS_MAPS和MB_MILLIS_MAPS）。 |
| reduce tasks总时间（MILLIS_REDUCES）         | 以毫秒为单位的reduce tasks运行总时间。包括推测执行的tasks。有对应的统计核心和内存使用时间的counters（VCORES_MILLIS_REDUCES和MB_MILLIS_REDUCES）。 |

#### 1.1.3、用户定义的Java Counters（User-Defined Java Counters）

MapReduce允许用户代码定义一些counters，它们如期望的那样在mapper或reducer中增加。这些counters通过java enum对象定义，服务于组的相关counters（每个enum对象相当于一个组）。job可以定义任意数量的enum，每个enum有任意数量的属性（fileds）。enum对象的名字是组名，enum的属性是counter名。counters是全局的：MapReduce框架跨maps和reduces聚合counters以在job结束时产生总计的值。

###### 例 9-1. Application to run the maximum temperature job，including counting missing and malformed fiedls and quality codes

```java
public class MaxTemperatureWithCounters extends Configured implements Tool {
  
  enum Temperature {
    MISSING,
    MALFORMED
  }
  
  static class MaxTemperatureMapperWithCounters
    extends Mapper<LongWritable, Text, Text, IntWritable> {
    
    private NcdcRecordParser parser = new NcdcRecordParser();
  
    @Override
    protected void map(LongWritable key, Text value, Context context)
        throws IOException, InterruptedException {
      
      parser.parse(value);
      if (parser.isValidTemperature()) {
        int airTemperature = parser.getAirTemperature();
        context.write(new Text(parser.getYear()),
            new IntWritable(airTemperature));
      } else if (parser.isMalformedTemperature()) {
        System.err.println("Ignoring possibly corrupt input: " + value);
        context.getCounter(Temperature.MALFORMED).increment(1);
      } else if (parser.isMissingTemperature()) {
        context.getCounter(Temperature.MISSING).increment(1);
      }
      
      // dynamic counter
      context.getCounter("TemperatureQuality", parser.getQuality()).increment(1);
    }
  }
  
  @Override
  public int run(String[] args) throws Exception {
    Job job = JobBuilder.parseInputAndOutput(this, getConf(), args);
    if (job == null) {
      return -1;
    }
    
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    job.setMapperClass(MaxTemperatureMapperWithCounters.class);
    job.setCombinerClass(MaxTemperatureReducer.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    return job.waitForCompletion(true) ? 0 : 1;
  }
  
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MaxTemperatureWithCounters(), args);
    System.exit(exitCode);
  }
}
```

运行这个程序：

```
% hadoop jar hadoop-examples.jar MaxTemperatureWithCounters \
  input/ncdc/all output-counters
```

通过job client输出的counters如下：

```
Air Temperature Records
  Malformed=3
  Missing=66136856
TemperatureQuality
  0=1
  1=973422173
  2=1246032
  4=10764500
  5=158291879
  6=40066
  9=66136858
```

注意，通过使用以enum命名（使用下划线作为嵌套类的分隔符）的资源包，温度相关的counters变得更具可读性——本例中*MaxTemperatureWithCounters_Temperature.properties*，包含了展示名称映射。

#### 1.1.4、动态counters（Dynamic counters）

动态counter不是通过Java enum定义的。Java enum的属性在编译时定义，使用enums不能动态创建新的counters。此处想要计数气温quality code（记录中每个气温quality code的数量），尽管格式规范（气温数据格式规范）定义了气温quality code的可能值（可以将可能值枚举），但是使用动态counter获取记录中的气温quality code实际值更方便。这是要使用的是**Context**对象的，使用字符串组名和字符counter名的如下方法：

```java
public Counter getCounter(String groupName, String counterName);
```

它与另一个方法：

```java
public Counter getCounter(Enum<?> counterName);
```

其实是等价的，因为Hadoop把枚举转换为字符串以通过RPC传送counters。Enums相比之下使用更简单，提供类型安全，并且适用于大多数jobs。对于少数需要动态地创建counters的场合，可以使用**String**接口。

#### 1.1.5、获取counters（Retrieving counters）

除了通过web UI和命令行（使用mapred job -counter），还可以使用Java API获取counter的值。尽管通常在job运行的结尾，counters稳定时，获取counters，也可以在job运行中获取counters。

###### 例 9-2. Application to calculate the proportion of records with missing temperature fields

```java
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.mapreduce.*;
import org.apache.hadoop.util.*;

public class MissingTemperatureFields extends Configured implements Tool {

  @Override
  public int run(String[] args) throws Exception {
    if (args.length != 1) {
      JobBuilder.printUsage(this, "<job ID>");
      return -1;
    }
    String jobID = args[0];
    Cluster cluster = new Cluster(getConf());
    Job job = cluster.getJob(JobID.forName(jobID));
    if (job == null) {
      System.err.printf("No job with ID %s found.\n", jobID);
      return -1;
    }
    if (!job.isComplete()) {
      System.err.printf("Job %s is not complete.\n", jobID);
      return -1;
    }

    Counters counters = job.getCounters();
    long missing = counters.findCounter(
        MaxTemperatureWithCounters.Temperature.MISSING).getValue();
    long total = counters.findCounter(TaskCounter.MAP_INPUT_RECORDS).getValue();

    System.out.printf("Records with missing temperature fields: %.2f%%\n",
        100.0 * missing / total);
    return 0;
  }
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MissingTemperatureFields(), args);
    System.exit(exitCode);
  }
}
```

首先，通过使用job ID和方法*getJob()*从集群获取**Job**对象。确认过job已经完成后，调用**Job**的*getCounters()*方法获取封装了job所有counters的**Counters**对象。**Counters**类提供了多种多样的获取counters名称和值的方法。

如下为运行和输出：

```
% hadoop jar hadoop-examples.jar MissingTemperatureFields job_1410450250506_0007
Records with missing temperature fields: 5.47%
```

#### 1.1.6、用户自定义流counters（User-Defined Streaming Counters）

流MapReduce程序可以通过发送特殊格式行到标准错误输出流增加counters，此时作为一种控制渠道。行的格式必须如下：

```
reporter:counter:group,counter,amount
```

如下Python代码片段展示了如何将“Temperature”组中“Missing” counter的值加1:

```python
sys.stderr.write("reporter:counter:Temperature,Missing,1\n")
```

类似地，通过如下格式发送状态信息：

```
reporter:status:message
```

