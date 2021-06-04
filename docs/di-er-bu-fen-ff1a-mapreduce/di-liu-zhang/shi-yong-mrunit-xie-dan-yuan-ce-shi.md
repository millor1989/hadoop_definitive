## 3.使用MRUnit写单元测试（Writing a Unit Test with MRUnit）

**MRUnit**是一个通过传递已知输入到mapper或者reducer并检验输出是否符合预期的测试库。MRUnit和标准测试执行框架（如Junit）一起使用，所以可以在一般开发环境运行MapReduce jobs的测试。

### 3.1、Mapper

###### 例 6-5 MaxTemperatureMapper单元测试

```java
import java.io.IOException;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mrunit.mapreduce.MapDriver;
import org.junit.*;

public class MaxTemperatureMapperTest {

  @Test
  public void parsesValidRecord() throws IOException, InterruptedException {
    Text value = new Text("0043011990999991950051518004+68750+023550FM-12+0382" +
                                  // Year ^^^^
        "99999V0203201N00261220001CN9999999N9-00111+99999999999");
                              // Temperature ^^^^^
    new MapDriver<LongWritable, Text, Text, IntWritable>()
      .withMapper(new MaxTemperatureMapper())
      .withInput(new LongWritable(0), value)
      .withOutput(new Text("1950"), new IntWritable(-11))
      .runTest();
  }
}
```

思路就是：传递一条天气记录作为mapper的输入，检验输出为要读取的年份和温度。

测试的mapper：

```java
public class MaxTemperatureMapper
  extends Mapper<LongWritable, Text, Text, IntWritable> {

  enum Temperature {
    MALFORMED
  }

  private NcdcRecordParser parser = new NcdcRecordParser();

  @Override
  public void map(LongWritable key, Text value, Context context)
      throws IOException, InterruptedException {

    parser.parse(value);
    if (parser.isValidTemperature()) {
      int airTemperature = parser.getAirTemperature();
      context.write(new Text(parser.getYear()), new IntWritable(airTemperature));
    } else if (parser.isMalformedTemperature()) {
      System.err.println("Ignoring possibly corrupt input: " + value);
      context.getCounter(Temperature.MALFORMED).increment(1);
    }
  }
}
```

解析天气记录的类：

```java
public class NcdcRecordParser {

  private static final int MISSING_TEMPERATURE = 9999;

  private String year;
  private int airTemperature;
  private boolean airTemperatureMalformed;
  private String quality;

  public void parse(String record) {
    year = record.substring(15, 19);
    airTemperatureMalformed = false;
    // Remove leading plus sign as parseInt doesn't like them (pre-Java 7)
    if (record.charAt(87) == '+') { 
      airTemperature = Integer.parseInt(record.substring(88, 92));
    } else if (record.charAt(87) == '-') {
      airTemperature = Integer.parseInt(record.substring(87, 92));
    } else {
      airTemperatureMalformed = true;
    }
    quality = record.substring(92, 93);
  }

  public void parse(Text record) {
    parse(record.toString());
  }

  public boolean isValidTemperature() {
    return !airTemperatureMalformed && airTemperature != MISSING_TEMPERATURE
        && quality.matches("[01459]");
  }

  public boolean isMalformedTemperature() {
    return airTemperatureMalformed;
  }

  public String getYear() {
    return year;
  }

  public int getAirTemperature() {
    return airTemperature;
  }
}
```

**MapDriver**可以用于检验零个、一个、或者多个输出记录，根据调用withOutput\(\)方法的次数来确定。

```java
  public void ignoresMissingTemperatureRecord() throws IOException,
      InterruptedException {
    Text value = new Text("0043011990999991950051518004+68750+023550FM-12+0382" +
                                  // Year ^^^^
        "99999V0203201N00261220001CN9999999N9+99991+99999999999");
                              // Temperature ^^^^^
    new MapDriver<LongWritable, Text, Text, IntWritable>()
      .withMapper(new MaxTemperatureMapper())
      .withInput(new LongWritable(0), value)
      .runTest();
  }
```

如上例，丢失温度数据的记录会被过滤，所以没有输出。

### 3.2、Reducer

使用ReduceDriver测试reducer的例子：

```java
  public void returnsMaximumIntegerInValues() throws IOException,
      InterruptedException {
    new ReduceDriver<Text, IntWritable, Text, IntWritable>()
      .withReducer(new MaxTemperatureReducer())
      .withInput(new Text("1950"),
          Arrays.asList(new IntWritable(10), new IntWritable(5)))
      .withOutput(new Text("1950"), new IntWritable(10))
      .runTest();
  }
```

测试的reducer：

```java
public class MaxTemperatureReducer
  extends Reducer<Text, IntWritable, Text, IntWritable> {

  @Override
  public void reduce(Text key, Iterable<IntWritable> values,
      Context context)
      throws IOException, InterruptedException {

    int maxValue = Integer.MIN_VALUE;
    for (IntWritable value : values) {
      maxValue = Math.max(maxValue, value.get());
    }
    context.write(key, new IntWritable(maxValue));
  }
}
```

## 4.本地性地在测试数据上运行（Running Locally on Test Data）

写一个job driver，并在开发机器上用一些测试数据运行它。

### 4.1、在本地的Job Runner中运行job（Running a Job in a Local Job Runner）

###### 例 6-10 Application to find the maximum temperature

```java
public class MaxTemperatureDriver extends Configured implements Tool {

  @Override
  public int run(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.printf("Usage: %s [generic options] <input> <output>\n",
          getClass().getSimpleName());
      ToolRunner.printGenericCommandUsage(System.err);
      return -1;
    }

    Job job = new Job(getConf(), "Max temperature");
    job.setJarByClass(getClass());

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    job.setMapperClass(MaxTemperatureMapper.class);
    job.setCombinerClass(MaxTemperatureReducer.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    return job.waitForCompletion(true) ? 0 : 1;
  }

  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MaxTemperatureDriver(), args);
    System.exit(exitCode);
  }
}
```

使用本地配置在命令行运行：

```
% export HADOOP_CLASSPATH=target/classes/
% hadoop v2.MaxTemperatureDriver -conf conf/hadoop-local.xml \
  input/ncdc/micro output
```

等价地，可以使用**GenericOptionsParser**提供的**-fs**（用指定的URI设置默认文件系统，是-D fs.defaultFS=_uri_的快捷方式）和**-jt**（设置YARN资源管理器， -D yarn.resourcemanager.address=_host:port_的快捷方式）选项：

```
% hadoop v2.MaxTemperatureDriver -fs file:/// -jt local input/ncdc/micro output
```

### 4.2、测试Driver（Testing the Driver）

MapReuduce应用实现**Tool**接口除了可以使用灵活的配置选项，还能通过注入任意的**Configuration**对象进行测试。

###### 例 6-11、A test for MaxTemperatureDriver that uses a local,in-process job runner

```java
public class MaxTemperatureDriverTest {

  public static class OutputLogFilter implements PathFilter {
    public boolean accept(Path path) {
      return !path.getName().startsWith("_");
    }
  }

//vv MaxTemperatureDriverTestV2
  @Test
  public void test() throws Exception {
    Configuration conf = new Configuration();
    conf.set("fs.defaultFS", "file:///");
    conf.set("mapreduce.framework.name", "local");
    conf.setInt("mapreduce.task.io.sort.mb", 1);

    Path input = new Path("input/ncdc/micro");
    Path output = new Path("output");

    FileSystem fs = FileSystem.getLocal(conf);
    fs.delete(output, true); // delete old output

    MaxTemperatureDriver driver = new MaxTemperatureDriver();
    driver.setConf(conf);

    int exitCode = driver.run(new String[] {
        input.toString(), output.toString() });
    assertThat(exitCode, is(0));

    checkOutput(conf, output);
  }
//^^ MaxTemperatureDriverTestV2

  private void checkOutput(Configuration conf, Path output) throws IOException {
    FileSystem fs = FileSystem.getLocal(conf);
    Path[] outputFiles = FileUtil.stat2Paths(
        fs.listStatus(output, new OutputLogFilter()));
    assertThat(outputFiles.length, is(1));

    BufferedReader actual = asBufferedReader(fs.open(outputFiles[0]));
    BufferedReader expected = asBufferedReader(
        getClass().getResourceAsStream("/expected.txt"));
    String expectedLine;
    while ((expectedLine = expected.readLine()) != null) {
      assertThat(actual.readLine(), is(expectedLine));
    }
    assertThat(actual.readLine(), nullValue());
    actual.close();
    expected.close();
  }

  private BufferedReader asBufferedReader(InputStream in) throws IOException {
    return new BufferedReader(new InputStreamReader(in));
  }
}
```

这个例子，明确地设置了_fs.defaultFS_和_mapreduce.framework.name_，所以它使用的是本地文件系统和本地job runner。

测试Driver的第二种方式是使用“最小”集群（“mini-” cluster）运行它。Hadoop有一系列的测试类，_**MiniDFSCluster**_、_**MiniMRCluster**_、和_**MiniYARNCluster**_，提供创建在进程中集群（in-process clusters）的程式化方法。与本地job runner不同，最小集群支持对HDFS，MapReduce，YARN的测试。牢记，最小集群中的nodemanager会为tasks的运行启动独立的JVMs，这会使debug变得更困难。

也可以在命令行运行最小集群：

```
% hadoop jar \
  $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-*-tests.jar \
  minicluster
```

最小集群在Hadoop自身的自动化测试工具中应用广泛，但是也可以用于测试用户代码。Hadoop的**ClusterMapReduceTestCase**抽象类提供了写这种测试的基础，使用它的_setUp\(\)_和_tearDown\(\)_方法处理启动停止在进程HDFS和YARN集群，并生成一个和它们起工作的合适的**Configuration**对象。子类只用提供HDFS中的数据（或许复制自本地文件），运行一个MapReduce job，并确认输出即可。如下为一个例子：

```java
public class MaxTemperatureDriverMiniTest extends ClusterMapReduceTestCase {

  public static class OutputLogFilter implements PathFilter {
    public boolean accept(Path path) {
      return !path.getName().startsWith("_");
    }
  }

  @Override
  protected void setUp() throws Exception {
    if (System.getProperty("test.build.data") == null) {
      System.setProperty("test.build.data", "/tmp");
    }
    if (System.getProperty("hadoop.log.dir") == null) {
      System.setProperty("hadoop.log.dir", "/tmp");
    }
    super.setUp();
  }

  // Not marked with @Test since ClusterMapReduceTestCase is a JUnit 3 test case
  public void test() throws Exception {
    Configuration conf = createJobConf();

    Path localInput = new Path("input/ncdc/micro");
    Path input = getInputDir();
    Path output = getOutputDir();

    // Copy input data into test HDFS
    getFileSystem().copyFromLocalFile(localInput, input);

    MaxTemperatureDriver driver = new MaxTemperatureDriver();
    driver.setConf(conf);

    int exitCode = driver.run(new String[] {
        input.toString(), output.toString() });
    assertThat(exitCode, is(0));

    // Check the output is as expected
    Path[] outputFiles = FileUtil.stat2Paths(
        getFileSystem().listStatus(output, new OutputLogFilter()));
    assertThat(outputFiles.length, is(1));

    InputStream in = getFileSystem().open(outputFiles[0]);
    BufferedReader reader = new BufferedReader(new InputStreamReader(in));
    assertThat(reader.readLine(), is("1949\t111"));
    assertThat(reader.readLine(), is("1950\t22"));
    assertThat(reader.readLine(), nullValue());
    reader.close();
  }
}
```



