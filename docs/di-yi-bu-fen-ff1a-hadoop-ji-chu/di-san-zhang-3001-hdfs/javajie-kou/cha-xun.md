### 5.2.4、目录（Directories）

_**FileSystem**_提供了一个创建目录的方法：

```java
public boolean mkdirs(Path f) throws IOException;
```

这个方法会创建所有必要的父目录，如果这些目录不存在，与java.io.File的mkdirs\(\)方法类似。如果目录创建成功，返回true。

通常，不用明确的创建目录，因为FileSystem的create方法在创建文件时可以创建目录。

### 5.2.5、查询文件系统（Querying the Filesystem）

#### 5.2.5.1、文件元数据：FileStatus（File metadata:FileStatus）

文件系统的一个重要特征是它的目录结构导航和获取它存储的文件和目录的信息的能力。_**FileStatus**_类封装了目录和文件的文件系统元数据，包括文件长度、块的大小、复本、修改时间、拥有者、权限信息。

_**FileSystem**_的_getFileStatus\(\)_方法提供了获取单个文件或目录的FileStatus对象的途径。

###### _例 3-5： 展示文件状态信息（Demonstrating file status information）_

```java
import static org.junit.Assert.*;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.lessThanOrEqualTo;

import java.io.*;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.hdfs.MiniDFSCluster;
import org.junit.*;

// vv ShowFileStatusTest
public class ShowFileStatusTest {

  private MiniDFSCluster cluster; // use an in-process HDFS cluster for testing
  private FileSystem fs;

  @Before
  public void setUp() throws IOException {
    Configuration conf = new Configuration();
    if (System.getProperty("test.build.data") == null) {
      System.setProperty("test.build.data", "/tmp");
    }
    cluster = new MiniDFSCluster.Builder(conf).build();
    fs = cluster.getFileSystem();
    OutputStream out = fs.create(new Path("/dir/file"));
    out.write("content".getBytes("UTF-8"));
    out.close();
  }

  @After
  public void tearDown() throws IOException {
    if (fs != null) { fs.close(); }
    if (cluster != null) { cluster.shutdown(); }
  }

  @Test(expected = FileNotFoundException.class)
  public void throwsFileNotFoundForNonExistentFile() throws IOException {
    fs.getFileStatus(new Path("no-such-file"));
  }

  @Test
  public void fileStatusForFile() throws IOException {
    Path file = new Path("/dir/file");
    FileStatus stat = fs.getFileStatus(file);
    assertThat(stat.getPath().toUri().getPath(), is("/dir/file"));
    assertThat(stat.isDirectory(), is(false));
    assertThat(stat.getLen(), is(7L));
    assertThat(stat.getModificationTime(),
        is(lessThanOrEqualTo(System.currentTimeMillis())));
    assertThat(stat.getReplication(), is((short) 1));
    assertThat(stat.getBlockSize(), is(128 * 1024 * 1024L));
    assertThat(stat.getOwner(), is(System.getProperty("user.name")));
    assertThat(stat.getGroup(), is("supergroup"));
    assertThat(stat.getPermission().toString(), is("rw-r--r--"));
  }

  @Test
  public void fileStatusForDirectory() throws IOException {
    Path dir = new Path("/dir");
    FileStatus stat = fs.getFileStatus(dir);
    assertThat(stat.getPath().toUri().getPath(), is("/dir"));
    assertThat(stat.isDirectory(), is(true));
    assertThat(stat.getLen(), is(0L));
    assertThat(stat.getModificationTime(),
        is(lessThanOrEqualTo(System.currentTimeMillis())));
    assertThat(stat.getReplication(), is((short) 0));
    assertThat(stat.getBlockSize(), is(0L));
    assertThat(stat.getOwner(), is(System.getProperty("user.name")));
    assertThat(stat.getGroup(), is("supergroup"));
    assertThat(stat.getPermission().toString(), is("rwxr-xr-x"));
  }

}
```

如果文件或者目录不存在，_getFileStatus\(\)_方法会抛出_**FileNotFoundException**_。

如果判断文件或目录是否存在，可以使用_**FileSystem**_的_exists\(\)_方法：

```
public boolean exists(Path f) throws IOException;
```

#### 5.2.5.2、列出文件（Listing files）

_**FileSystem**_的一系列_listStatus\(\)_方法可以列出一个目录的内容：

```java
public FileStatus[] listStatus(Path f) throws IOException;
public FileStatus[] listStatus(Path f, PathFilter filter) throws IOException;
public FileStatus[] listStatus(Path[] files) throws IOException;
public FileStatus[] listStatus(Path[] files, PathFilter filter) throws IOException;
```

参数举是目录时，返回的结果是0个或者多个_**FileStatus**_对象（代表目录中的一个目录或者文件）组成的数组；如果路径参数指向的是一个文件，则返回一个只有一个_**FileStatus**_对象的数组。

如果参数是路径数组，那么结果相当于每个路径对应的FileStatus对象元素组成的一个数组。

重载的方法中有一个PathFilter参数，作用是限制匹配的文件和目录。

Hadoop的_**FileUtil**_的_stat2Paths\(\)_方法可以把_**FileStatus**_对象数组转换为_**Path**_对象数组。

###### 例 3-6：列出一个Hadoop文件系统中路径集合的FileStatus

```java
public class ListStatus {

  public static void main(String[] args) throws Exception {
    String uri = args[0];
    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(uri), conf);

    Path[] paths = new Path[args.length];
    for (int i = 0; i < paths.length; i++) {
      paths[i] = new Path(args[i]);
    }

    FileStatus[] status = fs.listStatus(paths);
    Path[] listedPaths = FileUtil.stat2Paths(status);
    for (Path p : listedPaths) {
      System.out.println(p);
    }
  }
}
```

#### 5.2.5.3、文件模式（File Patterns）

在一个操作中处理一组文件是一个常见的需求。例如，一个用于日志处理的MapReduce job可能会处理存放在几个目录中的一个月的文件。相比起枚举每一个文件和目录来指定输入，用一个使用通配符的表达式来匹配多个文件是更加方便的，一种方法是通配符（_globbing_）。Hadoop提供两种FileSystem的方法来处理通配符（globs）：

```java
public FileStatus[] globStatus(Path pathPattern) throws IOException;
public FileStatus[] globStatus(Path pathPattern, PathFilter filter) throws IOException;
```

_globStatus\(\)_方法返回匹配路径模式的FileStatus对象数组，按照path排序（sort by path）。PathFilter是一个可选的参数，用于进一步限制匹配的结果。

Hadoop支持的通配符和Unix bash shell支持的一样：

###### 表 3-2.通配符及其含义

| Glob（通配符） | Matches（匹配规则） |
| :--- | :--- |
| \* | 匹配0个或者多个字符 |
| ? | 匹配一个字符 |
| \[ab\] | 匹配集合{a,b}中的一个字符 |
| \[^ab\] | 匹配不在集合{a,b}中的一个字符 |
| \[a-b\] | 匹配闭区间\[a,b\]中的一个字符，a应该小于等于b |
| \[^a-b\] | 匹配一个不再闭区间\[a,b\]中的字符 |
| {a,b} | 匹配表达式a或b |
| \c | 转义字符，匹配c当它是元字符（metacharacter）的时候 |

**例**，假如存在如下目录结构：

```
/
├── 2007/
│   └── 12/
│       ├── 30/
│       └── 31/
└── 2008/
    └── 01/
        ├── 01/
        └── 02/
```

则有通配符、扩展表如下：

| Glob | Expansion |
| :--- | :--- |
| /\* | /2007 /2008 |
| /\*/\* | /2007/12 /2008/01 |
| /\*/12/\* | /2007/12/30 /2007/12/31 |
| /200? | /2007 /2008 |
| /200\[78\] | /2007 /2008 |
| /200\[7-8\] | /2007 /2008 |
| /200\[^01234569\] | /2007 /2008 |
| /\*/\*/{31,01} | /2007/12/31 /2008/01/01 |
| /\*/\*3{0,1} | /2007/12/30 /2007/12/31 |
| /\*/{12/31,01/01} | /2007/12/31 /2008/01/01 |

#### 5.2.5.4、PathFilter

通配符模式（Glob patterns）可能有时在描述要访问的文件集合方面有些不够强大。例如，使用通配符不能够排出一个特定的文件。_**FileSystem**_的_listStatus\(\)_方法和_globStatus\(\)_方法使用了一个可选的参数_**PathFilter**_，可以程序控制匹配：

```java
package org.apache.hadoop.fs;

public interface PathFilter {
  boolean accept(Path path);
}
```

_**PathFilter**_和用于**Path**对象（不是_File_对象）的_**java.io.FileFilter**_相当。

###### 例 3-7、一个排出匹配正则表达式的路径的PathFilter

```java
public class RegexExcludePathFilter implements PathFilter {
  
  private final String regex;

  public RegexExcludePathFilter(String regex) {
    this.regex = regex;
  }

  public boolean accept(Path path) {
    return !path.toString().matches(regex);
  }
}
```

使用：

```java
// 返回结果 /2007/12/30
fs.globStatus(new Path("/2007/*/*"), new RegexExcludeFilter("^.*/2007/12/31$"))
```

Filters只能用于文件名，不能用于文件的属性。此外，Filters可以执行glob模式和正则表达式不能执行的匹配。

