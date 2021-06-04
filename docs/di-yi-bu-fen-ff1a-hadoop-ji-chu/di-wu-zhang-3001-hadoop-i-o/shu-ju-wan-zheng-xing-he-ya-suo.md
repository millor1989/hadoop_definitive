## 1、数据完整性（Data Integrity）

Hadoop的用户期望在存储和处理过程中没有数据丢失或损坏。尽管磁盘或网络上的每个I/O操作很少将错误引入自己正在读写的数据中，但是当流经系统的数据体量大到hadoop能够处理的一样大时，数据损坏出现的概率就很高。

通常在数据首次进入系统时计算它的**checksum（校验和）**，在它通过不可靠渠道传输时再次计算checksum，通过比较就能发现数据是否损坏。这种技术只能发现错误，不能修复数据（这是不使用低端硬件，一定要使用ECC内存的原因）。注意，也有可能是checksum错误，而不是数据错误，但是这是不大可能的，checksum和数据相比非常小。

常用的错误检测码是CRC-32（32位循环冗余校验），为任何大小的输入计算一个32位整数的checksum。Hadoop的ChecksumFileSystem使用CRC-32计算checksum，而HDFS使用一个更高效的变种CRC-32C。

### 1、HDFS中的数据完整性

HDFS为写入的所有数据计算checksum，默认在读取的时候验证checksum。对每一份_**dfs.bytes-per-checksum**_字节的数据分别计算checksum。默认是512字节，因为CRC-32C的checksum是4字节长，存储开销小于1%。

datanodes负责在保存数据和数据的checksum之前对数据进行验证。在收到客户端的数据或其它datanodes的备份数据时执行验证。客户端把数据写到datanodes pipeline，pipeline的最后一个datanode验证checksum。如果datanode发现错误，客户端会收到一个IOException的子类，这个异常应该以应用特定的方式处理（例如，重试）。

当客户端从datanodes读数据时，也会通过和存储在datanode上的checksum比较进行checksum的验证。每个datanode保存一个持久的checksum校验日志，所以它直到每个blocks最后验证的时间。当一个客户端成功验证一个block后，它会告诉datanode，datanode更新它的校验日志。保存这种校验数据对于检测损坏磁盘很有价值。

除了在客户端读取是验证block，每个datanode在后台运行一个_**DataBlockScanner**_定期校验datanode上的所有blocks。这是为了防止物理存储媒介中“bit rot”引起的损坏。

因为HDFS保存着blocks的复本，它可以通过复制好的复本来“治愈”损坏的block。工作的方式是，当客户端读取一个block发现了错误时，它在抛出_**ChecksumException**_之前把该block和它所处的datanode报告给namenode；namenode把这个block复本标记为损坏，这样它就不会引导客户端访问这个block复本，也不会尝试把这个复本复制到其它datanode；然后它调度一个在其它datanode上备份这个block的任务，这样，复本因子就能恢复到期望值；备份操作完成后，删除损坏的复本。

可以使用_**hadoop fs -checksum**_命令来查看一个文件的checksum，这有助于检查hdfs中的两个文件是否具有相同的内容，例如distcp所做的。

#### 1.1、LocalFileSystem

Hadoop的_**LocalFileSystem**_可以计算客户端侧的checksum。这意味着写一个名为_filename_的文件时，文件系统透明地创建一个隐藏文件_.filename.crc_，保存在存放每个文件checksum的目录中。checksum文件大小由_**file.bytes-per-checksum**_属性控制，默认值是512字节。checksum文件尺寸放在_**.crc**_文件的元数据中，所以即使配置的尺寸变化了也能正确读取checksum文件。读取文件时验证checksum，检测到错误时，_**LocalFileSystem**_抛出_ChecksumException_.

计算checksum开销很小（在Java中，它们是由native代码实现的），一半增加读写文件时间的百分之几。对大多数应用来说，为了数据的完整性这点开销是可接受的。但是，一般在底层文件系统本身支持checksum时，也可以关闭checksum；通过使用_**RawLocalFileSystem**_代替_**LocalFileSystem**_即可。如果要在应用中全局地关闭checksum，通过设置属性_**fs.file.impl**_为_**org.apache.hadoop.fs.RawLocalFileSystem**_来重新映射文件URI的实现就行了；或者，如果指向禁用一部分读取操作的checksum，可以直接创建一个RawLocalFileSystem实例，如下：

```java
Configuration conf = ...
FileSystem fs = new RawLocalFileSystem();
fs.initialize(null, conf);
```

#### 1.2、ChecksumFileSystem

**LocalFileSystem**使用**ChecksumFileSystem**来进行checksum计算，这个类使为其它文件系统增加checksum计算变得简单，**ChecksumFileSystem**只是对FileSystem进行了包装。一般使用方法如下：

```java
FileSystem rawFs = ...
FileSystem checksumedFs = new ChecksumFileSytem(rawFs);
```

底层的文件系统叫做_raw_文件系统，可以通过_**ChecksumFileSystem**_的_**getRawFileSystem\(\)**_方法获取。_**ChecksumFileSystem**_有一些和checksum协同工作的有用的方法，例如_**getChecksumFile\(\)**_可以获取任意文件的checksum文件的路径。其它请查看文档。

如果读取文件时_**ChecksumFileSystem**_检测到错误，它会调用_**reportChecksumFailure\(\)**_方法。默认实现什么也没做，但是_**LocalFileSystem**_把错误文件和它的checksum移动到同一设备的名为_**bad\_files**_的副目录（side directory）。管理员应该定期检查这些错误文件并进行处理。

## 2、压缩（Compression）

文件压缩有两个主要的好处：减少存储文件的空间，并且提高了网络和磁盘间数据传输的速度。处理大量数据时，这些好处都很明显，因此需要认真考虑Hadoop中压缩的使用。

###### 表 5-1、压缩格式汇总（A summary of compression formats）

| Compression format | Tool | Algorithm | Filename extension | Splittable |
| :--- | --- | --- | --- | --- |
| DEFLATE | N/A | DEFLATE | _.deflate_ | No |
| gzip | _gzip_ | DEFLATE | _.gz_ | No |
| bzip2 | _bzip2_ | bzip2 | _.bz2_ | Yes |
| LZO | _lzop_ | LZO | _.lzo_ | No |
| LZ4 | N/A | LZ4 | _.lz4_ | No |
| Snappy | N/A | Snappy | _.snappy_ | No |

其中**DEFLATE**是一个标准实现为_zlib_的压缩算法。没有可以用来生称DEFLATE格式的文件的命令行工具，通常使用_**gzip**_。（gzip文件格式是有额外的头和一个脚的DEFLATE。）_**.deflate**_文件扩展名只是一个Hadoop的约定。

如果在预处理步骤进行索引，LZO文件是可以分割的。

所有的压缩算法都在空间/时间上有权衡：更快的压缩和解压速度通常会造成节约的空间更小。**表5-1**中的工具一般通过提供9个不同选项，可以控制压缩时的这个权衡值：_**-1代表速度最优，-9代表空间最省**_。例如，下面命令使用最快的压缩方法创建一个压缩文件_**file.gz**_:

```shell
gzip -1 file
```

不同的压缩工具有不同的压缩特性。_**gzip**_是一个通用的（general-purpose）压缩工具，时间/空间的权衡的中间水平。_**bzip2**_相比gzip压缩更高效，但是更慢；_**bzip2**_的解压速度比它的压缩速度更快，但是仍然比其它格式更慢。另一方面，_**LZO**_、_**LZ4**_和_**Snappy**_，都是速度方面最佳，比gzip要快很多，但是压缩不高效。_**Snappy**_和_**LZ4**_都在解压速度方面显著地快于_**LZO**_。

表5-1中的“Splittable”指的是压缩格式是否支持切分（即，是否可以在流中的任意点seek寻道并且从某些点开始读取）。可切分压缩格式尤其适用于MapReduce。

### 2.1、Codecs

_**codec**_是压缩-解压缩算法的实现。在Hadoop中，codec由_**CompressionCodec**_接口的实现代表。所以，例如，GzipCodec封装了gzip的压缩和解压算法。

###### 表 5-2、Hadoop压缩codec（Hadoop compression codecs）

| Compression format | Hadoop CompressionCodec |
| --- | --- |
| DEFLATE | org.apache.hadoop.io.compress.DefaultCodec |
| gzip | org.apache.hadoop.io.compress.GzipCodec |
| bzip2 | org.apache.hadoop.io.compress.BZip2Codec |
| LZO | com.hadoop.compression.lzo.LzopCodec |
| LZ4 | org.apache.hadoop.io.compress.Lz4Codec |
| Snappy | org.apache.hadoop.io.compress.SnappyCodec |

LZO库是GPL执照，可能不包括在Apache的发布项目中，所以它的Hadoop codecs要另外地从Google（或者GitHub，包括bug修复和更多工具）下载。LzopCodec，和_**lzop**_工具兼容，是本质上的LZO格式加上额外headers，一般正是所需要的。也有针对纯净LZO格式的LzoCodec，使用_.lzo\_deflate_文件扩展名（与DEFLATE类似，是没有headers的gzip）。

#### 2.1.1、使用CompressionCodec进行流压缩和解压（Compressing and decompressing streams with CompressionCodec）

**CompressionCodec**有两个方法用来压缩和解压数据。使用_**createOutputStrem\(OutputStream out\)**_方法创建一个**CompressionOutputStream**（把未压缩的数据写到这里面，使它以压缩的形式写到底层流）来压缩写到输出流的数据。相反地，使用_**createInputStream\(InputStream in\)**_来获取一个**CompressionInputStream**（从底层流读取未压缩的数据）来解压从输入流读取的数据。

**CompressionOutputStream**和**CompressionInputStream**和_java.util.zip.DeflaterOutputStream_和_java.util.zip.DeflaterInputStream_类似，但是前两者能够重置底层的压缩器（compressor）和解压缩器（decompressor）。这对于压缩数据流的片段为分离的blocks的应用很重要，例如SequenceFile。

###### 例 5-1、一个压缩从标准输入读取的数据并把它写到标注输出的程序（A program to compress data read from standard input and write it to standard output）

```java
public class StreamCompressor {

  public static void main(String[] args) throws Exception {
    String codecClassname = args[0];
    Class<?> codecClass = Class.forName(codecClassname);
    Configuration conf = new Configuration();
    CompressionCodec codec = (CompressionCodec)
        ReflectionUtils.newInstance(codecClass, conf);

    CompressionOutputStream out = codec.createOutputStream(System.out);
    IOUtils.copyBytes(System.in, out, 4096, false);
    out.finish();
  }
}
```

这个应用需要CompressionCodec实现的全限定名为第一个命令行参数。使用ReflectionUtils来构建一个codec的新的实例，然后获取一个包装了_**System.out**_的**CompressionOutputStream**包装类。然后使用**IOUtils**的工具方法_copyBytes\(\)_把输入复制到输出，把输入通过**CompressionOutputStream**进行压缩。最后调用**CompressionOutputStream**的_finish\(\)_方法，这个方法告诉compressor停止往压缩流写数据，但是没有关闭流。下面命令，使用StreamCompressor程序（使用GzipCodec）压缩字符串“Text”，然后使用gunzip解压：

```shell
echo "Text" | hadoop StreamCompressor org.apache.hadoop.io.compress.GzipCodec \
  | gunzip -
```

执行结果输出：

```shell
Text
```

#### 2.1.2、使用CompressionCodecFactory推断CompressionCodec（Inferring CompressionCodecs using CompressionCodecFactory）

读取压缩文件时，通常可以通过文件名扩展名推断它使用的codec。

**CompressionCodecFactory**使用_**getCodec\(\)**_方法提供了一个把文件扩展名映射为CompressionCodec的方法：

###### 例 5-2、一个使用从文件扩展名推测出的codec解压一个压缩文件的程序（A program to decompress a compressed file using a codec inffered from the file‘s extension）

```java
public class FileDecompressor {

  public static void main(String[] args) throws Exception {
    String uri = args[0];
    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(uri), conf);

    Path inputPath = new Path(uri);
    CompressionCodecFactory factory = new CompressionCodecFactory(conf);
    CompressionCodec codec = factory.getCodec(inputPath);
    if (codec == null) {
      System.err.println("No codec found for " + uri);
      System.exit(1);
    }

    String outputUri =
      CompressionCodecFactory.removeSuffix(uri, codec.getDefaultExtension());

    InputStream in = null;
    OutputStream out = null;
    try {
      in = codec.createInputStream(fs.open(inputPath));
      out = fs.create(new Path(outputUri));
      IOUtils.copyBytes(in, out, conf);
    } finally {
      IOUtils.closeStream(in);
      IOUtils.closeStream(out);
    }
  }
}
```

一旦获得了codec，通常去掉文件的压缩格式后缀来构成输出文件名（通过**CompressionCodecFactory**的静态方法_removeSuffix\(\)_）。使用这个程序，文件名为_file.gz_的压缩文件就被解压为_file_：

```shell
hadoop FileDecompressor file.gz
```

**CompressionCodecFactory**可以加载**表5-2**中除了_LZO_以外的所有codecs，此外，也能加载配置属性_**io.compression.codecs**_中配置的codecs（**表 5-3**）。这个属性默认是空的，如果要注册自定义的codec，才需要修改它。每个codec对应它默认的文件名扩展，这样CompressionCodecFactory可以通过指定的扩展名在注册的codecs中找到一个匹配的codec。

###### 表 5-3、压缩codec属性（Compression codec properties）

| Property name | Type | Default value | Description |
| --- | --- | --- | --- |
| io.compression.codecs | Comma-separated class names |  | A list of additional CompressionCodec classes for compression/decompression |

#### 2.1.3、原生类库（Native libraries）

为了性能，最好使用原生类库进行压缩和解压。例如，在一项测试中，使用原生gzip类库（和使用内置的java实现相比）减少了50%的解压时间和10%的压缩时间。**表5-4**展示了每个压缩格式可用的Java和原生实现。所有的格式都有原生实现，但并不都有java实现（例如，LZO）

###### 表 5-4、压缩库实现（Compression library implementations）

| Compression format | Java implementation？ | Native implementation？ |
| --- | --- | --- |
| DEFLATE | Yes | Yes |
| gzip | Yes | Yes |
| bzip2 | Yes | Yes |
| LZO | No | Yes |
| LZ4 | No | Yes |
| Snappy | No | Yes |

Apache Hadoop的二进制压缩包（tarball）附带了为64位Linux预构建的原生压缩二进制代码，叫做_**libhadoop.so**_。对于其它平台，需要根据源码顶层的_**BUILDING.txt**_指导自己编译类库。

使用Java系统属性_**java.library.path**_指定原生类库。_**etc/hadoop**_目录中的_**hadoop**_脚本设置了这个属性，如果不使用这个脚本，那么需要在应用中设置这个属性。

默认情况下，Hadoop会为它运行平台上寻找原生类库，并且在找到的时候自动地加载。这意味着，为了使用原生类库不用改变任何设置。某些情况下，可能需要禁用原生类库，例如debug一个压缩相关的问题。通过设置_**io.native.lib.available**_为**false**，可以确保使用内置的Java类库（如果有的话）。

#### 2.1.4、CodecPool

如果在应用中使用了原生类库并且进行大量压缩和解压，请考虑使用_**CodecPool**_，它能够重用压缩器和解压缩器（compressors and decompressors），因而分摊了创建这些对象的开销。

###### 例 5-3、使用compressor池压缩和解压数据（A program to compress data read from standard input and write it to standard output using a pooled compressor）

```java
public class PooledStreamCompressor {

  public static void main(String[] args) throws Exception {
    String codecClassname = args[0];
    Class<?> codecClass = Class.forName(codecClassname);
    Configuration conf = new Configuration();
    CompressionCodec codec = (CompressionCodec)
      ReflectionUtils.newInstance(codecClass, conf);
    Compressor compressor = null;
    try {
      compressor = CodecPool.getCompressor(codec);
      CompressionOutputStream out =
        codec.createOutputStream(System.out, compressor);
      IOUtils.copyBytes(System.in, out, 4096, false);
      out.finish();
    } finally {
      CodecPool.returnCompressor(compressor);
    }
  }
}
```

从指定**CompressionCodec**的池中获取一个**Compressor**实例，这个实例可以用在codec的重载的方法_createOutputStream\(\)_中。使用finally代码块即使发生IOException也能确保compressor被返还给池子。

### 2.2、压缩和输入分片（Compression and Input Splits）

当考虑如何压缩要被MapReduce处理的数据时，了解压缩格式是否支持切分是很重要的。假如HDFS中保存着一个未压缩大小1G的文件，HDFS的block大小为128M，这个文件会被保存为8个blocks，使用这个文件的MapReduce job会创建8个输入分片（splits），每个分片作为于一个单独的map task的输入独立地进行处理。

如果这个文件是一个压缩尺寸为1G的_**gzip**_压缩的文件。和之前相同，HDFS会把它存在8个blocks中。可是，不会为每个block创建一个分片，因为在gzip流中不能从任意一点开始读取数据，因而map task不能独立于其它tasks读取它的分片。gzip格式使用DEFLATE保存压缩数据，DEFLATE把数据保存为一系列压缩的blocks。问题是每个block的开始是无法分辨的，所以读取时无法从数据流的任意当前位置前进到下一个block的起始位置读取下一个block从而实现于整个数据流同步。因此，_**gzip**_不支持切分。

在这种情况下，MapReduce会做正确的事情，不会尝试切分gzip压缩的文件，因为它知道输入数据是gzip压缩的（通过文件名扩展）并且gzip不支持切分。job能运行，但是会牺牲本地性：一个map会处理八个HDFS blocks，对map task来说多数数据不是本地的。更少的map，job粒度就会更大，耗时更长。

如果文件是**LZO**压缩文件，也会有相同的问题，因为底层压缩格式没有提供同步LZO流的方法。但是，可以使用带有Hadoop LZO类库的索引工具（indexer tool）对LZO文件进行预处理，可以从[Google](https://code.google.com/p/hadoop-gpl-compression/)和[GitHub](https://github.com/kevinweil/hadoop-lzo)网站获取LZO的类库Codec。这种工具构建一个分片点（split points）的索引，当使用合适的MapReduce输入格式时，可以有效地使它们可以切分。

另外，**bzip2**压缩文件，提供blocks间的同步标识（一个pi的48位近似值），所以它支持切分。

#### 2.2.1、应该使用哪种压缩格式（Which Compression format should i use？）

Hadoop应用处理巨大的数据集，所以应该尽量发挥压缩的优势。使用什么压缩格式要根据文件大小、格式、用于处理的工具而决定。以下是一些建议，大致按照效率由高到低排列：

- 使用容器文件格式，例如sequence文件、Avro数据文件、ORCFiles或者Parquet文件，所有这些文件都支持压缩和切分。快速的压缩器（LZO、LZ4或者Snappy）通常是好的选择。
- 使用支持切分的压缩格式，例如bzip2（尽管bzip非常慢），或者可以通过加索引支持切分的格式，如LZO。
- 在应用中把文件切分为块，使用任意格式压缩每一块的文件。这时，要注意块的大小，以便块大小接近HDFS的block的大小。
- 保存未压缩的文件。

对于巨大的文件，不应该使用不支持整个文件切分的压缩格式，因为失去本地性会让MapReduce应用变得效率低下。

### 2.3、在MapReduce中使用压缩（Using Compression in MapReduce）

当文件被MapReduce读取时，如果文件是压缩的，MapReduce会使用文件扩展名决定使用哪个codec，进而自动地解压文件。

为了压缩MapReduce job的输出，在job配置中设置***mapreduce.output.fileoutputformat.compress***属性为**true**并把***mapreduce.output.fileoutputformat.compress.codec***属性设置为使用的压缩codec的类名。另外也可以通过**FileOutputFormat**的方便的静态方法设置这两个属性，如**例5-4***所示：

```java
public class MaxTemperatureWithCompression {

  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperatureWithCompression <input path> " +
        "<output path>");
      System.exit(-1);
    }

    Job job = new Job();
    job.setJarByClass(MaxTemperature.class);

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    /*[*/FileOutputFormat.setCompressOutput(job, true);
    FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);/*]*/
    
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setCombinerClass(MaxTemperatureReducer.class);
    job.setReducerClass(MaxTemperatureReducer.class);
    
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

运行：

``` shell
hadoop MaxTemperatureWithCompression input/ncdc/sample.txt.gz output
```

最终输出的每个部分都是压缩的，解压一个部分：

```shell
gunzip -c output/part-r-00000.gz
```

输出：

1949    111
1950    22

如果要输出sequence files，可以通过属性***mapreduce.output.fileoutputformat.compress.type***设置使用的压缩类型。默认的是***RECORD***，即压缩单条记录。改为**BLOCK**，则压缩成组的记录，这种类型压缩更好推荐使用。**SequenceFileOutputFormat**有一个方便的静态方法*setOutputCompressionType()*也可以用来设置这个属性。

用于为MapReduce job输出设置压缩的配置属性，见**表5-5**。如果MapReduce驱动程序使用***Tool***接口，可以在命令行传递任意这些属性，这比起修改程序硬编码压缩属性更为方便。

###### 表 5-5、MapReduce压缩属性（MapReduce compression properties）

| Property name                                    | Type       | Default value                                   |
| :----------------------------------------------- | ---------- | :---------------------------------------------- |
| mapreduce.output.fileoutputformat.compress       | boolean    | false                                           |
| mapreduce.output.fileoutputformat.compress.codec | class name | org.apache.hadoop.io.compress.DefaultCodec |
| mapreduce.output.fileoutputformat.compress.type  | String     | RECORD                                          |

#### 2.3.1、压缩map输出（Compressing map output）

即使MapReduce应用读写的是未压缩的数据，它也将从压缩map阶段的中间输出结果受益。map输出是写到磁盘的并且通过网络传输到reducer节点，所以使用LZO、LZ4或者Snappy这些快速的压缩器可以获得简单地性能提升，因为传输数据的量减少了。开启map输出压缩和设置压缩格式的配置属性如**表5-6**:

###### 表 5-6、Map输出压缩属性（Map output compression properties）

| Property name                       | Type    | Default value                              | Description                                  |
| ----------------------------------- | ------- | ------------------------------------------ | -------------------------------------------- |
| mapreduce.map.output.compress       | boolean | false                                      | Whether to compress map outputs              |
| mapreduce.map.output.compress.codec | class   | org.apache.hadoop.io.compress.DefaultCodec | The compression codec to use for map outputs |

如下是开启job中gzip map输出压缩的代码：

```java
    Configuration conf = new Configuration();
    conf.setBoolean(Job.MAP_OUTPUT_COMPRESS, true);
    conf.setClass(Job.MAP_OUTPUT_COMPRESS_CODEC, GzipCodec.class,
        CompressionCodec.class);
    Job job = new Job(conf);
```

