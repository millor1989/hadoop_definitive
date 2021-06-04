### 5.1、从Hadoop URL读取数据（Reading Data from a Hadoop URL）

从Hadoop文件系统读取数据的最简单的方法是：使用_**java.net.URL**_对象打开一个流，并从流来读取数据。

首先要调用_**URL**_的_**setURLStreamHandlerFactory\(\)**_方法，设置参数为_**FsUrlStreamHandlerFactory**_实例。这样Java才能认识Hadoop的hdfs URL scheme。每个JVM只能调用这个方法一次，一般在静态代码块中执行这个方法；这个限制意味着如果程序的其他部分——也许是第三方组件——设置了URLStreamHandlerFactory，程序就不能使用这种方式从Hadoop读取数据。

```java
import java.io.InputStream;
import java.net.URL;

import org.apache.hadoop.fs.FsUrlStreamHandlerFactory;
import org.apache.hadoop.io.IOUtils;

public class URLCat {

  static {
    URL.setURLStreamHandlerFactory(new FsUrlStreamHandlerFactory());
  }

  public static void main(String[] args) throws Exception {
    InputStream in = null;
    try {
      in = new URL(args[0]).openStream();
      IOUtils.copyBytes(in, System.out, 4096, false);
    } finally {
      IOUtils.closeStream(in);
    }
  }
}
```

### 5.2、使用FileSystem API读取数据（Reading Data Using the FileSystem API）

如上节所述，如果为应用设置URLStreamHandlerFactory不可用。那么，可以使用FileSystem API来为一个文件打开一个输入流。

Hadoop文件系统中的一个文件由一个Hadoop _**Path**_对象代表（不是java.io.File对象，它被紧密的绑定到本地文件系统）。可以把_**Path**_当做Hadoop文件系统URI，例如：_hdfs://localhost/user/tom/quangle.txt_。

_**FileSystem**_是一个通用的文件系统API，所以第一步是从HDFS获取一个_**FileSystem**_的实例。获取_**FileSystem**_实例的静态工厂方法有：

```java
// 返回默认的文件系统（core-site.xml中指定的，如果core-site.xml没有指定，返回默认的本地文件系统）
public static FileSystem get(Configuration conf) throws IOException;
// 使用指定的URI和授权信息获取文件系统，如果没有指定URI的scheme则返回默认文件系统
public static FileSystem get(URI uri, Configuration conf) throws IOException;
// 获取给定用户的文件系统，在使用安全hadoop的场合很重要
public static FileSystem get(URI uri, Configuration conf, String user) throws IOException;
```

Configuration对象封装了一个客户端或者服务端的配置，通过读取classpath的配置文件（例如，_etc/hadoop/core-site.xml_）设置。

获取本地文件系统实例的简单方法是：

```java
public static LocalFileSystem getLocal(Configuration conf) throws IOException;
```

当有了_**FileSystem**_实例的时候，可以使用_**open\(\)**_方法获取某个文件的输入流：

```java
// 使用了一个4KB的默认缓存
public FSDataInputStream open(Path f) throws IOException;
public abstract FSDataInputStream open(Path f, int bufferSize) throws IOException;
```

使用FileSystem来展示文件内容：

```java
import java.io.InputStream;
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class FileSystemCat {

  public static void main(String[] args) throws Exception {
    String uri = args[0];
    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(uri), conf);
    InputStream in = null;
    try {
      in = fs.open(new Path(uri));
      IOUtils.copyBytes(in, System.out, 4096, false);
    } finally {
      IOUtils.closeStream(in);
    }
  }
}
```

#### 5.2.1、FSDataInputStream

_**FileSystem**_的_open\(\)_方法实际上返回的是_**FSDataInputStream**_而不是标准的_java.io_类。这个类是_java.io.DataInputStream_的一个特例，支持随机访问，可以从流的任意部分读取数据：

```java
package org.apache.hadoop.fs;

public class FSDataInputStream extends DataInputStream
    implements Seekable, PostionedReadable {
    // 实现省略
}
```

_**Seekable**_接口允许将指针移到文件的某个位置（_**seek\(long pos\)**_方法）并且提供一个查询指针距离文件开头的偏移量的方法（_**getPos\(\)**_）：

```java
public interface Seekable {
  void seek(long pos) throws IOException;
  long getPos() throws IOException;
}
```

使用_**seek\(\)**_方法时，如果位置比文件的长度大会抛出IOException；与java.io.InputStream的_**skip\(\)**_方法不同的是，_**skip\(\)**_方法只能将指针移到当前位置之后，而_**seek\(\)**_方法可以将指针移动到文件的任意的绝对位置。

_**seek\(\)**_方法的使用：

```java
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class FileSystemDoubleCat {

  public static void main(String[] args) throws Exception {
    String uri = args[0];
    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(uri), conf);
    FSDataInputStream in = null;
    try {
      in = fs.open(new Path(uri));
      IOUtils.copyBytes(in, System.out, 4096, false);
      // 返回文件流的开始位置
      in.seek(0); 
      IOUtils.copyBytes(in, System.out, 4096, false);
    } finally {
      IOUtils.closeStream(in);
    }
  }
}
```

_**PostionedReadable**_接口允许从文件指定的位置读取文件的一部分：

```java
public interface PositionedReadable {
  // 把文件中从offset位置开始，读取length 字节长度的数据到buffer。返回值是实际读取的字节数，可能会比length小。
  public int read(long position, byte[] buffer, int offset, int length) throws IOException;
  // 读取length长度的数据到buffer中
  public void readFully(long position, byte[] buffer, int offset, int length) throws IOException;
  // 读取buffer.length长度的数据到buffer中
  public void readFully(long position, byte[] buffer) throws IOException;
}
```

对于_**readFully**_方法，如果读取到了文件的结尾，会抛出_EOFException_。

所有的读取方法都保留了文件当前的偏移量（指针位置）并且是线程安全的（尽管_**FSDataInputStream**_不是为同时访问设计的；因此，最好还是创建多个实例），所以在读取文件的主体部分的时候，这些读取方法提供了访问其它部分（也许是，metadata）的便利方法。

最后，需要注意的是，调用_**seek\(\)**_方法是一个相对**开销高**的操作，不应该频繁使用。

#### 5.2.2、写数据（Writing Data）

_**FileSystem**_有一些**创建文件**的方法。最简单的是：

```java
// 使用Path对象创建文件，并返回输出流可以用来写文件，会覆盖已经存在的文件，默认的复本数量，缓存大小4KB
public FSDataOutputStream create(Path f) throws IOException;
// 指定是否覆盖已经存在的文件
public FSDataOutputStream create(Path f, boolean overwrite) throws IOException;
// 指定复本数量
public FSDataOutputStream create(Path f, short replication) throws IOException;
// 指定使用的缓存大小
public FSDataOutputStream create(Path f, boolean overwrite, int bufferSize) throws IOException;
// 指定Progressable回调接口
public FSDataOutputStream create(Path f, Progressable progress) throws IOException;
// 还有其它重载的创建方法 此处省略...
```

**WARNING**.FileSystem的create方法会创建任意不存在的目录。尽管方便，可能造成意外。如果希望目录不存在时，不创建文件，可以使用_**exists\(\)**_方法先判断目录是否存在。也可以使用_**FileContext**_，可以控制是否创建目录。

```java
package org.apache.hadoop.util;

public interface Progressable {
  /**
   * Report progress to the Hadoop framework.
   */
  public void progress();
}
```

_**Progressable**_，是一个回调接口，应用使用它可以被通知数据写到datanodes的进度，但是也只能简单的通知发生了一些事情，并不详细。**Progress**在MapReduce应用中很重要。在使用看下例：

###### 复制本地文件到Hadoop文件系统：

```java
public class FileCopyWithProgress {
  public static void main(String[] args) throws Exception {
    String localSrc = args[0];
    String dst = args[1];

    InputStream in = new BufferedInputStream(new FileInputStream(localSrc));

    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(dst), conf);
    OutputStream out = fs.create(new Path(dst), new Progressable() {
      public void progress() {
        System.out.print(".");
      }
    });

    IOUtils.copyBytes(in, out, 4096, true);
  }
}
```

可以使用_**append\(\)**_方法向已经存在的文件中追加数据（也有很多的重载方法）：

```java
public FSDataOutputStream append(Path f) throws IOException;
```

append操作允许一个用户往文件结尾追加数据。并非所有的Hadoop文件系统都支持追加操作。

在执行append时，本地集群的datanode总是报错，执行_hadoop fs -ls_ 命令也会报错，未解决：：

```
19/08/31 17:33:13 ERROR datanode.DataNode: Millor:50010:DataXceiver error proces
sing READ_BLOCK operation  src: /127.0.0.1:65126 dst: /127.0.0.1:50010
java.io.IOException: 远程主机强迫关闭了一个现有的连接。
        at sun.nio.ch.SocketDispatcher.read0(Native Method)
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:43)
        at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223)
        at sun.nio.ch.IOUtil.read(IOUtil.java:197)
        at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:380)
        at org.apache.hadoop.net.SocketInputStream$Reader.performIO(SocketInputS
tream.java:57)
        at org.apache.hadoop.net.SocketIOWithTimeout.doIO(SocketIOWithTimeout.ja
va:142)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:1
61)
        at org.apache.hadoop.net.SocketInputStream.read(SocketInputStream.java:1
31)
        at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
        at java.io.DataInputStream.readShort(DataInputStream.java:312)
        at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.readOp(Receiver
.java:58)
        at org.apache.hadoop.hdfs.server.datanode.DataXceiver.run(DataXceiver.ja
va:212)
        at java.lang.Thread.run(Thread.java:748)
```

hadoop默认貌似是不支持append操作的hdfs-site.xml配置_dfs.support.append_默认值是false：

```
<property>
  <name>dfs.support.append</name>
  <value>true</value>
  <description>Does HDFS allow appends to files?
               This is currently set to false because there are bugs in the
               "append code" and is not supported in any prodction cluster.
  </description>
</property>
```

append对hadoop貌似不是推荐的操作！一般删除再创建。

### 5.2.3、FSDataOutputStream

_**FileSystem**_的_create\(\)_方法返回一个**FSDataOutputStream**。有一个_**getPos\(\)**_方法查看文件当前指针位置。

FSDataOutputStream不支持seek。因为HDFS只允许顺序写一个打开的文件或者一个写过的文件。不支持，任意位置的写操作。

