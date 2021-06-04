## 3、序列化（Serialization）

**序列化**是为了网络传输或者为了写到持久化存储把结构化的对象转换为字节流的过程。**反序列化**是把字节流转换为一系列的结构化对象的反过程。

序列化用在分布式数据处理的两个截然不同的领域：进程间通信（interprocess communication）和持久化存储（persistent storage）。

在Hadoop中，系统中的节点之间的进程间通信是用RPCs实现的。RPC协议使用序列化将信息转换称二进制流发送到远端节点，远端节点把二进制流反序列化为原始信息。通常，期望的序列化格式为：

- **紧凑（Compact）**

  紧凑的格式最能充分利用网络带宽，网络带宽是数据中心最稀缺的资源。

- **快速（Fast）**

  进程间通信是分布式系统的支柱，所以序列化和反序列化有尽可能小的性能开销是决定性的。

- **可扩展（Extensible）**

  为了满足需求协议随时会变，所以对客户端和服务器来说应该能够简单地以可控的方式改变协议。例如，可以为方法调用调用增加一个新的参数并让新的服务器能够从旧的客户端以旧的格式（没有新加的参数）接收信息。

- **支持互操作（Interoperable）**通用性、可互通性

  对某些系统来说，期望服务端能够支持以不同语言写成客户端，所以序列化格式需要被设计得能够跨语言的。

从表面上看，用于持久化存储的数据格式和序列化框架有不同的需求。毕竟，一个RPC的寿命短于疫苗，而持久化的数据可能在保存数年后读取。但是结果是，RPC序列化的四个需求属性对于持久化存储格式也是至关重要的。期望存储格式是紧凑的（有效使用存储空间），快速的（读写TB级数据的开销最小），可扩展的（可以透明地读取以就格式保存的数据），支持互操作（可以使用不同的语言读写持久化数据）。

Hadoop使用自己的序列化格式，Writables，必然是紧凑、快速的，但是不容易扩展或者使用除了Java以外的语言。因为，Writables是Hadoop的核心（大部分MapReduce程序使用它们作为key\value的类型）。

### 3.1、Writable接口（The Writable Interface）

**Writable**接口定义了两个方法，一个用于把它的状态写到一个**DataOutput**二进制流，一个用于从一个**DataInput**二进制流读取它的状态：

``` java
package org.apache.hadoop.io;

import java.io.DataOutput;
import java.io.DataInput;
import java.io.IOException;

public interface Writable {
  /** 
   * Serialize the fields of this object to <code>out</code>.
   * 
   * @param out <code>DataOuput</code> to serialize this object into.
   * @throws IOException
   */
  void write(DataOutput out) throws IOException;

  /** 
   * Deserialize the fields of this object from <code>in</code>.  
   * 
   * <p>For efficiency, implementations should attempt to re-use storage in the 
   * existing object where possible.</p>
   * 
   * @param in <code>DataInput</code> to deseriablize this object from.
   * @throws IOException
   */
  void readFields(DataInput in) throws IOException;
}
```

**IntWritable**，Java int的包装。创建并设置值：

```java
IntWritable writable = new IntWritable();
writable.set(163);
```

等价地，使用构造器创建：

```java
IntWritable writable = new IntWritable(163);
```

为了测试IntWritable的序列化格式，写一个小的帮助方法，把java.io.ByteArrayOutputStream包装在java.io.DataOutputStream（一个java.io.DataOutput的实现）中来获取序列化流中的字节：

```java
public static byte[] serialize(Writable writable) throws IOException {
	ByteArrayOutputStream out = new ByteArrayOutputStream();
	DataOutputStream dataOut = new DataOutputStream(out);
	writable.write(dataOut);
	dataOut.close();
	return out.toByteArray();
}
```

一个integer是4字节：

```java
IntWritable writable = new IntWritable(163);
byte[] bytes = serialize(writable);
assertThat(bytes.length, is(4));
```

这些字节以大端（big-endian）优先的顺序写入（最重要的字节首先写入流，这是由java.io.DataOutput接口决定的），使用Hadoop的**StringUtils**的方法可以查看它们对应的十六进制数：

``` java
assertThat(StringUtils.byteToHexString(bytes), is("000000a3"));
```

测试反序列化。创建一个从字节数组读取**Writable**对象的帮助方法：

```java
public static byte[] deserialize(Writable writable, byte[] bytes) throws IOException {
	ByteArrayInputStream in = new ByteArrayInputStream(bytes);
	DataInputStream dataIn = new DataInputStream(in);
	writable.readFields(dataIn);
	dataIn.close();
	return bytes;
}
```

构造一个新的、没有值的IntWritable，然后调用deserialize()从前例输出的数据读取值。然后，通过使用***get()***方法，验证它的值与原始值相同：

```java
IntWritable newWritable = new IntWritable();
deserialize(newWritable, bytes);
assertThat(newWritable.get(), is(163));
```

#### 3.1.1、WritableComparable 和 comparators

IntWritable实现了**WritableComparable**接口，它是Writable和java.lang.Comparable接口的子接口：

```java
package org.apache.hadoop.io;

public interface WritableComparable<T> extends Writable,Comparable<T> {
}
```

对于MapReduce来说类型比较是至关重要的，在排序阶段（sorting phase）keys之间要进行相互的比较。Hadoop提供的一个优化是Java的Comparator的扩展**RawComparator**：

```java
package org.apache.hadoop.io;

import java.util.Comparator;

public interface RawComparator<T> extends Comparator<T> {
    public int compare(byte[] b1, int s1, int l1, byte[] b2, int s2, int l2);
}
```

这个接口让实现者能够在比较从数据流读取的未反序列化为对象的记录，因而避免了对象创建的开销。例如，IntWritable的comparator实现了RawComparator接口的compare()方法，通过分别从字节数组b1、b2的指定位置（s1,s2）指定长度（l1,l2）读取一个integer并直接进行比较。

**WritableComparator**是一个针对**WrtiableComaprable**类的通用的**RawComparator**的实现。它提供了两个主要的方法。第一个是，原始的comapre()方法，从数据流反序列化出要比较的对象，并调用对象的compare()方法。第二个是，充当RawComparator实例的工厂（Writable实现已经注册）。例如，为了获取IntWritable的comparator，只需要：

```java
RawComparator<IntWritable> comparator = WritableComparator.get(IntWritable.class);
```

这个comparator可以用来比较两个IntWritable对象（第一个方法）：

```java
IntWritable w1 = new IntWritable(163);
IntWritable w2 = new IntWritable(67);
assertThat(comparator.compare(w1, w2), greaterThan(0));
```

也可以用来比较IntWritable的序列化表示（第二个方法）：

``` java
byte[] b1 = serialize(w1);
byte[] b2 = serialize(w2);
assertThat(comparator.compare(b1, 0, b1.length, b2, 0, b2.length), greaterThan(0));
```

### 3.2、Writable类

Hadoop有大量的Writable类，位于org.apache.hadoop.io包中。

###### 图5-1、Writable类层级（Writable class hierarchy）

![](/assets/1568173190359.png)

#### 3.2.1、Java基本类型的Writable包装类（Writable wrappers for Java primitives）

###### 表 5-7、Java基本类型的Writable包装类

| Java primitive | Writable implementation | Serialized size（bytes） |
| -------------- | ----------------------- | ------------------------ |
| boolean        | BooleanWritable         | 1                        |
| byte           | ByteWritable            | 1                        |
| short          | ShortWritable           | 2                        |
| int            | IntWritable             | 4                        |
|                | VIntWritable            | 1-5                      |
| float          | FloatWritable           | 4                        |
| long           | LongWritable            | 8                        |
|                | VLongWritable           | 1-9                      |
| double         | DoubleWritable          | 8                        |

除了**char**，所有的java基本类型都有对应的Writable包装类，char可以保存在IntWritable中。都有get()、set()方法用于获取和保存包装的值。

编码整数时可以选择定长格式和变长格式。如果值足够小，变长格式只使用小到一个字节；否则它们使用第一个字节来表示值是正值还是负值并且有后面有多少字节。例如，163需要两个字节：

```java
byte[] data = serialize(new VIntWritable(163));
assertThat(StringUtils.byteToHexString(data), is("8fa3"))
```

定长编码和变长编码如何选择呢？当数值在整个值空间中分布非常均匀，例如使用精心设计的hash函数时。大多数数字值趋向于非均匀分布，一般而言，编程编码节省空间。变长编码的另一个优势是，可以把VIntWritable转换为VLongWritable，因为它们的编码事实上是一样的。所以，使用变长表示，能够有增长空间，不用在一开始就用8字节的long表示。

#### 3.2.2、Text

**Text**是UTF-8序列的**Writable**包装类。可以被当作是java.lang.String的Writable类。

**Text**类使用一个int（变长编码）来保存字符串编码中的字节数量，最大值是2G。此外，Text使用标识的UTF-8，使它具有了和其它理解UTF-8的工具互通的潜能。它的序列化格式为表示字符串UTF-8编码字节数的VInt，跟着字符串UTF-8编码的字节。

##### 3.2.2.1、索引（Indexing）

因为**Text**的重点在使用标准UTF-8，它和java的String类型有些不同。**Text**类的索引是编码的字节序列的位置而言的，不是字符串的Unicode字符，也不是Java的**String**类的char *code unit*。对于ASCII字符串，这三个索引位置概念是相同的。如下为**Text**的*charAt()*方法的使用例子：

```java
Text t = new Text("hadoop");
assertThat(t.getLength(), is(6));
assertThat(t.getBytes().length, is(6));
assertThat(t.charAt(2), is((int) 'd'));
assertThat("Out of bounds", t.charAt(2), is(-1));
```

注意charAt()方法返回的是一个代表一个Unicode *code point*的int值，与Java的String的变种方法charAt()返回**char**类型字符不同。**Text**有一个*find()*方法与Java String的indexOf()方法类似：

```java
Text t = new Text("hadoop");
assertThat("Find a subString", t.find("do"), is(2));
assertThat("Find first 'o'", t.find("o"), is(3));
assertThat("Find 'o' from position 4 or later", t.find("o", 4), is(4));
assertThat("No match", t.find("pig"), is(-1));
```

##### 3.2.2.2、Unicode

当使用编码为多于一个字节的字符时，**Text**和**String**的区别就明显了。比如，**表5-8**展示的Unicode字符：

###### 表 5-8 Unicode字符

| Unicode code point      | U+0041                 | U+00DF                     | U+6671                      | U+10400                      |
| ----------------------- | ---------------------- | -------------------------- | --------------------------- | ---------------------------- |
| **Name**                | LATIN CAPITAL LETTER A | LATIN SMALL LETTER SHARP S | 晱(a unified Han ideograph) | DESERT CAPITAL LETTER LONG I |
| **UTF-8 code units**    | 41                     | c3 9f                      | e6 9d b1                    | f0 90 90 80                  |
| **Java representation** | \u0041                 | \u00DF                     | \u6771                      | \uD801\uDC00                 |

表中的字符除了U+10400，都可以用一个Java **char**类型表示。U+10400是一个补充字符用两个Java **char**表示，被称为代理对（*surrogate pair*）。**例5-5**展示了处理**表5-8**这四个字符组成的字符串时，String和Text的不同：

###### 例 5-5、String和Text类的不同

```java
public class StringTextComparisonText {

	@Test
	public void string() throws UnsupportedEncodingException {

		String s = "\u0041\u00DF\u6771\uD801\uDC00";
		assertThat(s.length(), is(5));// Unicode字符长度
		assertThat(s.getBytes("UTF-8").length, is(10));// UTF-8编码字节长度

		assertThat(s.indexOf("\u0041"), is(0));// Unicode字符索引
		assertThat(s.indexOf("\u00DF"), is(1));
		assertThat(s.indexOf("\u6771"), is(2));
		assertThat(s.indexOf("\uD801\uDC00"), is(3));

		assertThat(s.charAt(0), is('\u0041'));// char code unit
		assertThat(s.charAt(1), is('\u00DF'));
		assertThat(s.charAt(2), is('\u6771'));
		assertThat(s.charAt(3), is('\uD801'));
		assertThat(s.charAt(4), is('\uDC00'));

		assertThat(s.codePointAt(0), is(0x0041));// Unicode字符的code point
		assertThat(s.codePointAt(1), is(0x00DF));
		assertThat(s.codePointAt(2), is(0x6771));
		assertThat(s.codePointAt(3), is(0x10400));
	}

	@Test
	public void text() {

		Text t = new Text("\u0041\u00DF\u6771\uD801\uDC00");
		assertThat(t.getLength(), is(10));// UTF-8编码字节长度

		assertThat(t.find("\u0041"), is(0));// UTF-8编码字节索引
		assertThat(t.find("\u00DF"), is(1));
		assertThat(t.find("\u6771"), is(3));
		assertThat(t.find("\uD801\uDC00"), is(6));

        // 通过UTF-8编码字节索引获取对应Unicode字符的code point
		assertThat(t.charAt(0), is(0x0041));
		assertThat(t.charAt(1), is(0x00DF));
		assertThat(t.charAt(3), is(0x6771));
		assertThat(t.charAt(6), is(0x10400));
	}
}
```

通过测试证实，**String**的长度是它所包含的**char** code unit的个数，而**Text**对象的长度是它的UTF-8编码字节的数量。类似地，**String**的***indexOf()***方法返回的是**char** code unit的索引，而**Text**的***find()***方法返回的是对应字节的偏移量。

**String**的*charAt()*方法返回指定索引对应的**char** code unit，但是在有代理对情况下，**char** code unit不能代表一个完整的Unicode字符。**String**的*codePointAt()*方法，通过字符串中的Unicode字符索引，获取一个代表Unicode字符的int值。事实上，**Text**的*charAt()*方法更像**String**的*codePointAt()*而不是String的同名方法；不同的一点是，**Text**的*charAt()*通过字节偏移量获取代表对应Unicode字符的int值。

##### 3.2.2.3、遍历（Iteration）

通过使用字节偏移量作为索引来遍历**Text**中的Unicode字符是复杂的，因为不能简单的索引加一来遍历。遍历**Text**的习惯（idiom）有些模糊（obscure）：把**Text**对象转换为*java.nio.ByteBuffer*，然后用转换后的buffer重复调用**Text**的静态方法*bytesToCodePoint()*；这个方法以**int**形式获取下一个code point并且更新buffer中的指针位置；当*bytesToCodePoint()*返回-1时，达到字符串的结尾。如例5-6所示：

```java
public class TextIterator {
  
  public static void main(String[] args) {    
    Text t = new Text("\u0041\u00DF\u6771\uD801\uDC00");
    
    ByteBuffer buf = ByteBuffer.wrap(t.getBytes(), 0, t.getLength());
    int cp;
    while (buf.hasRemaining() && (cp = Text.bytesToCodePoint(buf)) != -1) {
      System.out.println(Integer.toHexString(cp));
    }
  }
}
```

执行结果：

41
df
6771
10400

##### 3.2.2.4、可变性（Mutability）

**Text**与**String**的另一个不同是，**Text**是可变的（与Hadoop的所有Writable实现一样，除了NullWritable，它是单例的）。可以通过调用**Text**的*set()*方法重用**Text**实例。例如：

```java
Text t = new Text("Hadoop");
t.set("pig");
assertThat(t.getLength(), is(3));
assertThat(t.getBytes().length, is(3));
```

###### 3.2.2.4.1、Text的*getLength()*

再某些情况下，通过*getBytes()*方法获取的字节数组的长度可能大于getLength()方法的返回值：

```java
Text t = new Text("Hadoop");
t.set(new Text("pig"));
assertThat(t.getLength(), is(3));
assertThat("Byte length not shortened", t.getBytes().length, is(6));
```

所以，在使用**Text**的*getBytes()*方法时要使用它的*getLength()*方法来确定取得的**字节数组**中有多少的**有效数据**。

##### 3.2.2.5、转换为String（Resorting to String）

**Text**没有**java.lang.String**那样丰富的操作字符串的API，所以许多情况下需要把**Text**转换为**String**。使用Text的toString()方法即可：

```java
assertThat(new Text("hadoop").toString(), is("hadoop"));
```

#### 3.2.3、BytesWritable

**BytesWritable**是一个二进制数据的数组的包装。它的序列化格式是用一个4字节整数区域指定字节的数量，跟着是存储的字节。例如，长度为2，元素为3和5的字节数组被序列化为一个4字节整数（00000002）跟着两个字节（03和05）:

```java
BytesWritable b = new BytesWritable(new byte[] {3, 5});
byte[] bytes = serialize(b);
assertThat(StringUtils.byteToHexString(bytes), is("000000020305"));
```

**BytesWritable**是可变的，通过使用它的*set()*方法可改变它的值。与**Text**类似，**BytesWritable**的getBytes()方法返回的字节数组的长度——容量——可能不能反映保存再**BytesWritable**的数据的真实长度。可以通过**BytesWritable**的*getLength()*方法确定字节的数量：

```java
b.setCapacity(11);
assertThat(b.getLength(), is(2));
assertThat(b.getBytes().length, is(11));
```

#### 3.2.4、NullWritable

**NullWritable**是一个Writable的特殊类型，它的序列化长度是0。 它不从流读数据也不往流写数据。它充当占位符；例如，在MapReduce中不使用的key或value可以声明为**NullWritable**，高效地存储一个常量空值。在**SequenceFile**中如果希望存储一系列的值，与键值对相对应，**NullWritable**也可以用作**SequenceFile**的key。**NullWritable**是一个不可变的单例，可以通过**NullWritable**.*get()*获取它的实例。

```java
NullWritable writable = NullWritable.get();
assertThat(serialize(writable).length, is(0));
```

#### 3.2.5、ObjectWritable and GenericWritable

**ObjectWritable**是一个用于（Java基本类型，**String**，**enum**，**Writable**，**null**或者任意这些类型数据）通用包装类。用于Hadoop RPC中进行方法参数、返回结果类型的封装和解封。

当一个属性可能是多种类型时**ObjectWritable**很有用。例如，**SequenceFile**中的值有多种类型，就可以声明值型为**ObjectWritable**并把各种类型抱在**ObjectWritable**中。作为一个通用机制，它浪费了大量的空间，因为当它每次被序列化时都会写包装的类型的类名。如果类型种类的数量很小并且事先知道，可以通过一个类型的静态数组并且使用这个数组的索引作为类型的序列化引用来改善性能；这是GenericWritable使用的方法，必需继承它来指定支持的类型。

```java
    ObjectWritable src = new ObjectWritable(Integer.TYPE, 163);
    ObjectWritable dest = new ObjectWritable();
    ReflectionUtils.copy(new Configuration(), src, dest);// 复制
    assertThat((Integer) dest.get(), is(163));
```

```java
public class BinaryOrTextWritable extends GenericWritable {
  private static Class[] TYPES = { BytesWritable.class, Text.class };

  @Override
  @SuppressWarnings("unchecked")
  protected Class<? extends Writable>[] getTypes() {
    return TYPES;
  }
}
```

#### 3.2.6、Writable集合（Writable collections）

org.apache.hadoop.io包包含了6种**Writable**集合类型：**ArrayWritable**、**ArrayPrimitiveWritable**、**TwoDArrayWritable**、**MapWritable**、**SortedMapWritable**、**EnumSetWritable**。

**ArrayWritable**和**TwoDArrayWritable**分别是**Writable**实例的数组和二维数组。**ArrayWritable**和**TwoDArrayWritable**的所有这些元素必须是同一类（class）的实例，通过构造器指定类：

``` java
ArrayWritable writable = new ArrayWritable(Text.class);
```

在把Writable集合作为类型的场合，例如SequenceFile的键、值，或者MapReduce的输入，则需要继承**ArrayWritable**或**TwoDArrayWritable**以静态地设置类型。例如：

```java
public class TextArrayWritable extends ArrayWritable {
  public TextArrayWritable() {
    super(Text.class);
  }
}
```

**ArrayWritable**或**TwoDArrayWritable**都有*get()*和*set()*方法，也有*toArray()*方法用来创建一个数组或者二维数组的前拷贝。

**ArrayPrimitiveWritable**是java基本类型数组的一个包装。在调用set()方法时检测组件类型，所以不用通过子类来设置类型。

**MapWritable**是**java.util.Map&lt;Writable, Writable&gt;**的一个实现，**SortedMapWritable**是**java.util.SortedMap&lt;Writable, Writable&gt;**的一个实现。每个键/值字段使用的类型是响应字段序列化格式的一部分。类型存储为单个字节（充当类型数组的索引）。类型数组使用org.apache.hadoop.io包中标准类型填充，但是自定义的Writable类型也可以使用（通过为非标准类型写一个编码类型数组头）。根据它们的实现，**MapWritable**和**SortedMapWritable**为自定义类型使用正**byte**值，所以在任何**MapWritable**和**SortedMapWritable**实例中最多可以使用127种不同的非标准**Writable**类。使用不同类型作为**MapWritable**的键和值的例子：

```java
    MapWritable src = new MapWritable();
    src.put(new IntWritable(1), new Text("cat"));
    src.put(new VIntWritable(2), new LongWritable(163));
    
    MapWritable dest = new MapWritable();
    ReflectionUtils.copy(new Configuration(), src, dest);
    assertThat((Text) dest.get(new IntWritable(1)), is(new Text("cat")));
    assertThat((LongWritable) dest.get(new VIntWritable(2)),
        is(new LongWritable(163)));
```

没有set和list的Writable集合实现。一般的set可以通过使用值为**NullWritable**的**MapWritable**（或者对应与sorted set 使用**SortedMapWritable**）模仿。还有enum的sets对应的**EnumSetWritable**。对于单一**Writable**类型的list，**ArrayWritable**是合乎要求的，但是对于一个包含不同**Writable**类型list，可以通过使用**GenericWritable**来包装**ArrayWritable**中的元素。又或者使用MapWritable的思想写一个通用的ListWritable。

