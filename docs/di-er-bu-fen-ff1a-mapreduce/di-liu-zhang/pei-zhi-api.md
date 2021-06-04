## 1、配置API（The Configuration API）

Hadoop组件由Hadoop的配置API进行配置。一个**Configuration**类的实例代表一组配置属性和它们对应的值。每个属性都是**String**类型的名字，值的类型可以是boolean、int、long、float、String、Class和`java.io.File`或者String的集合。

`Configurations`实例从源文件读取它的属性。

###### 例 6-1 一个简单的配置文件，_configuration-1.xml_

```xml
<?xml version="1.0"?>
<configuration>
  <property>
    <name>color</name>
    <value>yellow</value>
    <description>Color</description>
  </property>

  <property>
    <name>size</name>
    <value>10</value>
    <description>Size</description>
  </property>

  <property>
    <name>weight</name>
    <value>heavy</value>
    <final>true</final>
    <description>Weight</description>
  </property>

  <property>
    <name>size-weight</name>
    <value>${size},${weight}</value>
    <description>Size and weight</description>
  </property>
</configuration>
```

配置访问：

```java
    Configuration conf = new Configuration();
    conf.addResource("configuration-1.xml");
    assertThat(conf.get("color"), is("yellow"));
    assertThat(conf.getInt("size", 0), is(10));
    assertThat(conf.get("breadth", "wide"), is("wide"));
```

需要注意的是，XML文件中没有保存类型信息；可以在读取属性时指定属性类型。**Configuration**的**get\(\)**方法可以指定默认值。

### 1.1、合并配置源文件（Combining Resources）

###### 例 6-2，第二个配置文件，configuration-2.xml

```xml
<?xml version="1.0"?>
<configuration>
  <property>
    <name>size</name>
    <value>12</value>
  </property>

  <property>
    <name>weight</name>
    <value>light</value>
  </property>
</configuration>
```

按顺序添加配置文件：

```java
    Configuration conf = new Configuration();
    conf.addResource("configuration-1.xml");
    conf.addResource("configuration-2.xml");
```

定义在后添加的配置文件中的属性会覆盖较早的定义。但是标记为**final**的属性不会被覆盖，通常尝试覆盖final属性会导致配置错误，所以，在有尝试覆盖final属性时，会有警告信息记入日志。

```java
    assertThat(conf.getInt("size", 0), is(12));
    assertThat(conf.get("weight"), is("heavy"));
```

### 1.2、变量扩展（Variable Expansion）

配置属性可以用其它属性或者系统属性来定义。例如_configuration-1.xml_中的_size-weight_属性。

```java
    assertThat(conf.get("size-weight"), is("12,heavy"));
    System.setProperty("size", "14");
    // 配置文件中使用了${size}，会优先用系统属性
    assertThat(conf.get("size-weight"), is("14,heavy"));
```

系统属性的优先级要比定义在资源文件中的属性优先级高。这个特征在命令行使用-D_property=value_ JVM参数时很有用。

注意，尽管可以用系统属性定义配置属性，除非系统属性使用配置属性进行了再定义（比如${size}），否则无法通过configuration API访问系统属性。

```java
    System.setProperty("length", "2");
    assertThat(conf.get("length"), is((String) null));
```

## 2、配置开发环境（Setting Up the Development Environment）

###### 例 6-3.用于构建和测试MapReduce应用的Maven POM

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.hadoopbook</groupId>
  <artifactId>hadoop-book-mr-dev</artifactId>
  <version>4.0</version>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.version>2.5.1</hadoop.version>
  </properties>
  <dependencies>
    <!-- Hadoop main client artifact -->
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-client</artifactId>
      <version>${hadoop.version}</version>
    </dependency>
    <!-- Unit test artifacts -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.mrunit</groupId>
      <artifactId>mrunit</artifactId>
      <version>1.1.0</version>
      <classifier>hadoop2</classifier>
      <scope>test</scope>
    </dependency>
    <!-- Hadoop test artifact for running mini clusters -->
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-minicluster</artifactId>
      <version>${hadoop.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <finalName>hadoop-examples</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <source>1.6</source>
          <target>1.6</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.5</version>
        <configuration>
          <outputDirectory>${basedir}</outputDirectory>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

只需要_**hadoop-client**_依赖就能构建MapReduce jobs，它包括了Hadoop客户端侧需要的和HDFS和MapReduce交互的类。使用_**junit**_进行单元测试，使用_**mrunit**_进行MapReduce测试。_**hadoop-minicluster**_库包含了“最小”集群在使用在单个JVM中运行的Hadoop集群进行测试时是很有用的。

### 2.1、管理配置（Managing Configuration）

开发Hadoop应用的时候，通常要在本地运行应用和集群运行应用之间切换。事实上，可能会有几个工作集群，或者有一个本地的“伪分布式”集群用于测试。

```java
hadoop fs -conf conf/hadoop-localhost.xml -ls .
```

_**-conf**_命令行选项来配置使用哪个配置文件。如果忽略了_**-conf**_选项，则从_**$HADOOP\_HOME**_根目录下的_**etc/hadoop**_子目录获取Hadoop配置。或者，如果设置了_**HADOOP\_CONF\_DIR**_，则从这个位置获取配置。

### 2.2、GenericOptionsParser，Tool，and ToolRunner

Hadoop带了一些帮助类让从命令行运行jobs变得更简单。**GenericOptionParser**解析通用Hadoop命令行选项，并把它们设置到应用的**Configuration**对象中。通常不直接使用**GenericOptionsParser**，因为实现**Tool**接口并用**ToolRunner**运行应用更加方便，**ToolRunner**内部使用了**GenericOptionsParser**。

```java
public interface Tool extends Configurable {

  int run(String [] args) throws Exception;
}
```

**Tool**接口继承了**Configurable**接口，**Configured**类实现了**Configurable**接口。

###### 例 6-4 An example Tool implementation for printing the properties in a Configuration

```java
public class ConfigurationPrinter extends Configured implements Tool {

  static {
    Configuration.addDefaultResource("hdfs-default.xml");
    Configuration.addDefaultResource("hdfs-site.xml");
    Configuration.addDefaultResource("yarn-default.xml");
    Configuration.addDefaultResource("yarn-site.xml");
    Configuration.addDefaultResource("mapred-default.xml");
    Configuration.addDefaultResource("mapred-site.xml");
  }

  @Override
  public int run(String[] args) throws Exception {
    Configuration conf = getConf();
    for (Entry<String, String> entry: conf) {
      System.out.printf("%s=%s\n", entry.getKey(), entry.getValue());
    }
    return 0;
  }

  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new ConfigurationPrinter(), args);
    System.exit(exitCode);
  }
}
```

ConfigurationPrinter的main方法没有直接调用它的run\(\)方法，而是调用了**ToolRunner**的静态的run方法，这个run方法执行的是实现了**Tool**接口的ConfigurationPrinter对象的run方法。**ToolRunner**也使用了**GenericOptionsParser**来获取通过命令行指定的标准选项，并把它们设置在**Configuration**实例上。

也可以使用GenericOptionsParser设置私有的属性，例如：

```
% hadoop ConfigurationPrinter -D color=yellow | grep color
# 运行结果
color=yellow
```

使用**-D**指定的属性优先级高于配置文件中的对应属性。例如，通过**-D mapreduce.job.reduces=**_**n**_（JVM -D选项设置，-D后面不能有空格），可以覆盖集群设置的或者客户端侧配置文件设置的reducers的数量。

