### 3.3、实现一个自定义的Writable（Implementing a Custom Writable）

Hadoop有一组自带的**Writable**实现服务于大多数情况；偶尔，也有要自定义Writable实现的情况。使用自定义Writable，可以完全控制二进制表示和排序顺序。因为Writables是MapReduce数据路径的核心，调试二进制表示可以显著影响性能。Hadoop自带的Writable实现类型都是调试的很好的，但是对于复杂的结构，最好是创建新的Writable类型而不是使用自带的类型拼凑。

**忠告**：如果要考虑写一个自定义的Writable，也许值得尝试其它的序列化框架，比如**Avro**，可以以声明的方式定义自定义的类型。参考第12章。

**例 5-7**、展示了如何创建一个自定义的Writable：

###### 例 5-7 、一个存储一对Text对象的Writable实现

```java
import java.io.*;

import org.apache.hadoop.io.*;

public class TextPair implements WritableComparable<TextPair> {

  private Text first;
  private Text second;
  
  public TextPair() {
    set(new Text(), new Text());
  }
  
  public TextPair(String first, String second) {
    set(new Text(first), new Text(second));
  }
  
  public TextPair(Text first, Text second) {
    set(first, second);
  }
  
  public void set(Text first, Text second) {
    this.first = first;
    this.second = second;
  }
  
  public Text getFirst() {
    return first;
  }

  public Text getSecond() {
    return second;
  }

  @Override
  public void write(DataOutput out) throws IOException {
    first.write(out);
    second.write(out);
  }

  @Override
  public void readFields(DataInput in) throws IOException {
    first.readFields(in);
    second.readFields(in);
  }
  
  @Override
  public int hashCode() {
    return first.hashCode() * 163 + second.hashCode();
  }
  
  @Override
  public boolean equals(Object o) {
    if (o instanceof TextPair) {
      TextPair tp = (TextPair) o;
      return first.equals(tp.first) && second.equals(tp.second);
    }
    return false;
  }

  @Override
  public String toString() {
    return first + "\t" + second;
  }
  
  @Override
  public int compareTo(TextPair tp) {
    int cmp = first.compareTo(tp.first);
    if (cmp != 0) {
      return cmp;
    }
    return second.compareTo(tp.second);
  }
}
```

实现的第一部分是直观的：有两个**Text**变量，*first*和*second*，以及相关的构造器和*getter*、*setter*。所有的**Writable**实现必须有一个默认的构造器以便MapReduce框架可以实例化然后通过调用*readFields()*方法填充字段值。**Writable**实例是可变的并且经常重用，所以应该避免在*write()*方法或者*readFields()*方法中分配对象。

TextPair的*write()*方法反过来通过委派序列化给Text对象的*write()*方法来序列化每个Text字段对象到输出流。类似地，*readFields()*方法委派Text对象的*readFields()*方法来反序列化输入流中的字节。**DataOutput**和**DataInput**接口有丰富的用于序列化和反序列化Java基本类型的方法，所以通常，可以完全控制**Wrtiable**对象的线上格式（wire format）。

正如在Java中编写类的时候一样，编写**Writable**实现，需要重写继承自java.lang.Object的*hashCode()*，*equals()*，*toString()*方法。**HashPartitioner**（MapReduce中的默认partitioner）使用*hashCode()*方法来选择reduce分区。所以要写一个好的hash函数恰当的确保每个reduce分区都有相似的大小。

**警告：**如果要和**TextOutputFormat**一起使用自定义的Writable，必须实现它的*toString()*方法。**TextOutputFormat**会为输出的表示在keys和values上使用*toString()*方法。

TextPair是一个**WritableComparable**的实现，它提供了*compareTo()*方法的实现，可以根据需要排序：先按第一个字符串排序，再按照第二个字符串排序。注意，除了它可以保存的Text对象的数量，TextPair与**TextArrayWritable**不同，因为**TextArrayWritable**只是一个**Writable**，不是**CompareWritable**。

#### 3.3.1、为了速度实现一个RawComparator（Implementing a RawComparator for speed）

对**例5-7**还有可以进行优化的地方。例如，当TextPair用作MapReduce的key时，它必须反序列化为一个对象来使用compareTo()方法。所以可以考虑使用不进行反序列化的比较。

TextPair是两个Text对象的一个串联，Text对象的二进制表示为一个表示这个字符串所包含的UTF-8字节数的**VInt**，后面跟着这些字符串的UTF-8字节。实现RawComparator比较的方法是，通过读取VInt，这样就知道了第一个**Text**的字符串的字节数，进而可以得到第一个**Text**的序列化表示的对应的字节数；使用合适的偏移量可以委托**Text**对象的**RawComparator**进行比较。如**例 5-8**所示：

###### 例 5-8、比较TextPair字节表示的RawComparator（A RawComparator for comparing TextPair byte representations）

```java
  public static class Comparator extends WritableComparator {
    
    private static final Text.Comparator TEXT_COMPARATOR = new Text.Comparator();
    
    public Comparator() {
      super(TextPair.class);
    }

    @Override
    public int compare(byte[] b1, int s1, int l1,
                       byte[] b2, int s2, int l2) {
      try {
        /* TextPair的第一个Text的序列化长度（VInt占用字节数大小 + VInt）
         * VInt为字符串对应的UTF-8编码字节的数量
         */
        int firstL1 = WritableUtils.decodeVIntSize(b1[s1]) + readVInt(b1, s1);
        int firstL2 = WritableUtils.decodeVIntSize(b2[s2]) + readVInt(b2, s2);
        // 比较第一个Text的序列化字节数组
        int cmp = TEXT_COMPARATOR.compare(b1, s1, firstL1, b2, s2, firstL2);
        if (cmp != 0) {
          return cmp;
        }
        // 比较第二个Text的序列化字节数组
        return TEXT_COMPARATOR.compare(b1, s1 + firstL1, l1 - firstL1,
                                       b2, s2 + firstL2, l2 - firstL2);
      } catch (IOException e) {
        throw new IllegalArgumentException(e);
      }
    }
  }

  static {
    // 注册comparator
    WritableComparator.define(TextPair.class, new Comparator());
  }
```

实际上是继承了**WritableComparator**而不是直接实现**RawComparator**，因为它提供一些方便的方法和默认的实现。

静态代码块**注册**了为TextPair自定义的raw comparator，以便当MapReduce使用TextPair时知道使用raw comparator作为**默认**的comparator。

**WritableComparator**使用：

``` java
TextPair tp1 = new TextPair("ab", "b");
TextPair tp2 = new TextPair("ab", "c");
assertThat(WritableComparator.get(TextPair.class).compare(tp1, tp2), is(-1));
```

#### 3.3.2、自定义comparators（Custom comparators）

前例中可以看出，编写raw comparators需要处理一些字节级别的细节。如果需要编写自定义的raw comparator查看*org.apache.hadoop.io*包中**Writable**的一些实现来获取更多的主意是值得的。WritableUtils中的工具方法也是很方便的。

如果可能，自定义的comparators也应该编写为**RawCompartators**。这些comparators实现了不同于默认comparator自然排序的排序。如**例5-9**，展示了TextPair的一个comparator，**FirstComparator**，它只比较TextPair的第一个字符串；它也重写了使用两个对象作为参数的*compare()*方法，这样两个compare()方法就有相同的语义（semantics）。

###### 例 5-9、比较TextPair第一个字段的字节表示的自定义RawComparator（A  custom RawComparator for comparing the first field of TextPair byte representations)

```java
  public static class FirstComparator extends WritableComparator {
    
    private static final Text.Comparator TEXT_COMPARATOR = new Text.Comparator();
    
    public FirstComparator() {
      super(TextPair.class);
    }

    @Override
    public int compare(byte[] b1, int s1, int l1,
                       byte[] b2, int s2, int l2) {
      
      try {
        int firstL1 = WritableUtils.decodeVIntSize(b1[s1]) + readVInt(b1, s1);
        int firstL2 = WritableUtils.decodeVIntSize(b2[s2]) + readVInt(b2, s2);
        return TEXT_COMPARATOR.compare(b1, s1, firstL1, b2, s2, firstL2);
      } catch (IOException e) {
        throw new IllegalArgumentException(e);
      }
    }
    
    @Override
    public int compare(WritableComparable a, WritableComparable b) {
      if (a instanceof TextPair && b instanceof TextPair) {
        return ((TextPair) a).first.compareTo(((TextPair) b).first);
      }
      return super.compare(a, b);
    }
  }
```

### 3.4、序列化框架（Serialization Frameworks）

尽管大多数MapReduce程序使用Writable作为key和value类型，但这不是MapReduce API的要求。事实上，可以使用任何类型，唯一的要求是有一个类型的二进制表示进行转换的机制。

为了支持这，Hadoop有一个用于可拔插序列化框架的API。每个序列化框架用一个**Serialization**实现（在org.apache.hadoop.io.serializer包中）表示。例如，**WritableSerialization**，是用于**Writable**类型的**Serialization**实现。

**Serialization**定义了类型和**Serializer**实例（把对象转为字节流）和**DeSerializer**实例（把字节流转为对象）的映射。

通过设置***io.serializations***属性为逗号分隔的类名集合来注册**Serialization**实现。它的默认值包括*org.apache.hadoop.io.serializer.WritableSerialization*和*Avro Specific and Reflect serializations*，所以只有**Writable**和**Avro**对象可以开箱即用的序列化和反序列化。

Hadoop包括一个使用Java对象序列化的**JavaSerialization**。尽管它让在MapReduce程序中使用Java标准类型例如**Integer**和**String**变得方便，但是**Java Object Serialization**不像**Writables**那样高效，所以不值得作这种权衡。

为什么不使用Java Object Serialization？Java自己的序列化机制Java Object Serialization（简称Java Serialization），是和Java语言紧密绑定的，不能满足Hadoop系统compact、fast、extensible、interoperable的要求。

#### 3.4.1、序列化IDL（SerializationIDL）

有些其它序列化框架以不同的方式解决这个问题：不是通过代码定义类型，而是以语言中立（language-neutral 语言无关）声明的形式使用**IDL**（*interface description language*）定义。这个系统可以为不同语言生成类型，有益于通用性（interoperability）。它们一般定义不同版本的scheme使类型改进直观。

**Apache Thrift**和**Google Protocol Buffers**都是流行的序列化框架，都是是常用的二进制数据持久化格式。把它们用作MapReduce格式仅有有限的支持；可是它们内在地作为Hadoop的一部分用于RPC和数据交换。

**Avro**是一种基于IDL的序列化框架，非常适用于Hadoop大规模数据处理。