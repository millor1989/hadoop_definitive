# 5、HiveQL

###### 表17-2.A high-level comparison of SQL and HiveQL

| Feature | SQL | HiveQL |
| --- | --- | --- |
| Updates | UPDATE，INSERT，DELETE | UPDATE，INSERT，DELETE |
| Transactions | 支持 | 部分支持 |
| Indexes | 支持 | 支持 |
| Data Types | Integral，floating-point，fixed-point，text and binary string，temporal | Boolean，integral，floating-point，fixed-point，text and binary strings，temporal，array，map，struct |
| Functions | 内置函数 | 内置函数 |
| Multitable inserts | 不支持 | 支持 |
| CREATE TABLE...AS SELECT | SQL-92中无效，某些数据库支持 | 支持 |
| SELECT | SQL-92 | SQL-92。用于部分排序的_SORT BY_，_LIMIT_用来限制返回行数 |
| Joins | SQL-92，或者变种（连接_FROM_子句中的表，连接条件位于_WHERE_子句中） | Inner joins，outer joins，semi joins，map joins，cross joins |
| Subqueries | 在任意子句中（相关或者不相关） | 在_FROM_，_WHERE_，或者_HAVING_子句中（不支持不相关的子查询） |
| Views | Updatable（materialized or nonmaterialized） | Read-only（materialized views not supported） |
| Extension points | User-defined functions，stored procedures | User-defined functions，MapReduce scripts |

### 5.1、Data Types

#### 5.1.1、Primitive types

BOOLEAN，TINYINT，SMALLINT，INT，BIGINT，FLOAT，DOUBLE，DECIMAL，STRING，VARCHAR，CHAR，BINARY，TIMESTAMP（精度纳秒），DATE（年月日）

#### 5.1.2、Complex types

ARRAY，MAP，STRUCT（封装了一组有名字的属性），UNION（数据类型选择，值必须匹配一个选项）；负载类型的声明必须指定其中的属性的类型（使用&lt;&gt;）,如下：

```sql
CREATE TABLE complex (
  c1 ARRAY<INT>,
  c2 MAP<STRING, INT>,
  c3 STRUCT<a:STRING, b:INT, c:DOUBLE>,
  c4 UNIONTYPE<STRING, INT>
);
```

复杂类型查询：

```
hive> SELECT c1[0], c2['b'], c3.c, c4 FROM complex;
1    2    1.0    {1:63}
```

### 5.2、Operators and Functions

\|\|：等价于 _OR_

_concat_：字符串连接

查看Hive的函数：

```sql
SHOW FUNCTIONS;
```

使用_DESCRIBE_命令获取函数使用说明：

```
hive> DESCRIBE FUNCTION length;
length(str | binary) - Returns the length of str or number of bytes in binary
 data
```

#### 5.2.1、Conversions

隐式转换（implicit conversion）规则：任何数值转换类型可以转换为更宽的类型，或者转为文本类型。任何文本类型可以隐式地转换为其它文本类型，也可以转换为DOUBLE或者DECIMAL。BOOLEAN不能隐式地转换为其它任何类型。TIMESTAMP和DATE可以隐式地转换为文本类型。

可以使用_CAST_执行明确地类型转换。但是，如果转换失败，_CAST_表达式返回_NULL_。

## 6、Tables

### 6.1、Managed Tables and External Tables

在Hive中创建表时，默认Hive管理数据，意味着，Hive把数据移动到表的仓库目录中。此外，也可以创建外部表（external table）。

当往managed table加载数据时，数据被移动到Hive的仓库目录。例如：

```sql
CREATE TABLE managed_table (dummy STRING);
LOAD DATA INPATH '/user/tom/data.txt' INTO table managed_table;
```

会把文件_hdfs://user/tom/data.txt_移动到表managed\_table的仓库目录中，即目录_hdfs://user/hive/warehouse/managed\_table_。需要注意的是，只有源文件和目标文件在同一个文件系统中移动才会成功。如果_LOAD_命令使用了_LOCAL_关键字，Hive会把本地文件系统中的文件复制到仓库目录，其它情况下LOAD可以被视为是移动操作。

删除表：

```sql
DROP TABLE managed_table;
```

表，表的元数据和表中的数据，都会被删除。因为_LOAD_命令执行了一个移动操作，_DROP_命令执行一个删除操作，数据将不复存在。这就是Hive内部表管理数据的意味所在。

对于外部表：

```sql
CREATE EXTERNAL TABLE external_table (dummy STRING)
  LOCATION '/user/tom/external_table';
LOAD DATA INPATH '/user/tom/data.txt' INTO TABLE external_table;
```

Hive不会去管理数据，加载数据时不会把数据移动到它的仓库目录。在创建外部表时，不会校验外部的位置是否存在。这是一个有用的特征，因为这意味着可以在创建表后延迟地创建数据。

删除外部表时，Hive不会处理表数据，只会删除表的元数据。

创建managed表也可以使用_LOCATION_关键字来指定表的位置：

```sql
CREATE TABLE `d_u_b_p`(
  `user_id` bigint, 
  `channel_id` bigint, 
  `module_id` bigint)
PARTITIONED BY ( 
  `s_app_key` string, 
  `p_day` string)
ROW FORMAT SERDE 
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://nameservice/g/big/db/big.db/d_u_b_p';
```

### 6.2、Partitions and Buckets

Hive把表组织为分区——一种根据分区字段的值（例如，日期）将表划分为粗粒度部分的方式。使用分区使针对部分数据的查询更快。

表或者分区可以被进一步划分为_buckets_，以给需要被更加高效地查询的数据额外的结构。

**※** 在metastore中有单独保存分区元数据的表（当然也有保存数据库元数据、表元数据的表）。

#### 6.2.1、Partitions

分区可以减少特定分区查询时扫描的数据量，提高特定分区查询的速度。表可以多维度分区。

分区在创建表的时候定义：

```sql
CREATE TABLE logs (ts BIGINT, line STRING)
PARTITIONED BY (dt STRING, country STRING);
```

把数据加载到分区表时，要明确地指定分区：

```sql
LOAD DATA LOCAL INPATH 'input/hive/partitions/file1'
INTO TABLE logs
PARTITION (dt='2001-01-01', country='GB');
```

在_PARTITION BY_子句中的字段也是表字段，叫作分区字段（partition columns）；但是，表的数据文件不包含这些字段，因为这些字段值派生子目录名称。在_SELECT_语句中使用分区字段时，Hive会执行输入修剪（_input pruning_）只扫描相关的分区。

查看表的分区：

```sql
hive> SHOW PARTITIONS logs;
dt=2001-01-01/country=GB
dt=2001-01-01/country=US
dt=2001-01-02/country=GB
dt=2001-01-02/country=US
```

#### 6.2.2、Buckets

将表或者分区组织为buckets的优点：1、使查询更加高效；2、使抽样更加高效。

bucketing给表额外的结构，Hive可以使用这种结构执行某些查询。特别是，进行join的两个表都是在相同的字段上进行bucketed——其中包含join字段——会被高效地实现为一个map-side join。

处理大型数据集时，在开发或者优化数据集时使用bucket对数据集的一部分（一定比例）进行（抽样）查询很方便。首先，使用_CLUSTERED BY_子句指定要进行bucket的字段，和buckets的数量：

```sql
CREATE TABLE bucketed_users (id INT, name STRING)
CLUSTERED BY (id) INTO 4 BUCKETS;
```

这里使用用户ID来决定bucket（which Hive does by hashing the value and reducing modulo the number of buckets），每个bucket都会高效地拥有一组随机的用户。

在map-side join的情况下，两个表以同样的方式进行bucket，处理左表的一个bucket的mapper知道右表对应的行在对应的bucket中，所以它只用获取那个bucket（只是右表所有数据的一小部分）进行join即可。这个优化在两个表的buckets数量是彼此的倍数时也有用；两个表不必有完全相同的buckets数量。连接两个进行了bucket的表的HiveQL详见[Map joins]()章节。

bucket中的数据还可以按照一个或者多个列进行排序。这可以使map-side joins更加高效，因为每个bucket的join编程了一个efficient merge sort。声明一个排序的进行了bucket的表的语法如下：

```sql
CREATE TABLE bucketed_users (id INT, name STRING)
CLUSTERED BY (id) SORTED BY (id ASC) INTO 4 BUCKETS;
```

怎样确保表中的数据是进行了bucket的呢？尽管可以把Hive外生成的数据加载到一个bucketed 表，通常让Hive从一个已经存在的表进行bucket更简单。

**警告**：Hive不会检查磁盘上的数据文件和表定义的buckets（数量上或者进行bucket的列）是否一致。如果不匹配，可能会在查询的时候得到一个错误或者未定义的结果。因此，建议让Hive来进行bucket。

假如有一个未进行bucket的表_users_：

```shell
hive> SELECT * FROM users;
0    Nat
2    Joe
3    Kay
4    Ann
```

如果要用这表的数据装配进行了bucket的表，需要设置_hive.enforce.bucketing_属性为**true**，以使Hive知道表定义中声明的buckets的数量。接下来，只要执行_INSERT_命令就行了：

```sql
INSERT OVERWRITE TABLE bucketed_users
SELECT * FROM users;
```

物理上，每个bucket只是表或者分区目录中的一个文件。文件名不重要，但是当按照字典式（lexicographic）顺序排列时，bucket _**n**_是第_**n**_个文件。事实上，buckets对应于MapReduce输出文件分区：一个job会产生和reduce tasks数量相同的buckets（输出文件）。通过查看_bucketed\_users_表的目录格式，便明白了。运行命令：

```shell
hive> dfs -ls /user/hive/warehouse/bucketed_users;
```

可看到，创建了四个文件，名称如下（名称是Hive生成的）：

```
000000_0
000001_0
000002_0
000003_0
```

第一个bucket包好IDs为0和4的用户，因为对于一个**INT** hash是这个整数本身，值是减去对buckets数4取模的结果（reduced modulo the number of buckets——4），这种情况下：

```shell
hive> dfs -cat /user/hive/warehouse/bucketed_users/000000_0;
0Nat
4Ann
```

使用**TABLESAMPLE**子句对表进行抽样（sample）可以看到相同的结果，这把查询限制为对表的一小部分的buckets进行，而不是整个表：

```sql
hive> SELECT * FROM bucketed_users
    > TABLESAMPLE(BUCKET 1 OUT OF 4 ON id);
4    Ann
0    Nat
```

bucket标号是基于1的，所以这个查询从四个buckets中的第一个获取所有的用户。对于一个大的、均匀分布的数据集，表所有数据的将近四分之一数据会被返回。也可以通过指定一个不同的比例（不必是buckets数量的精确的倍数，因为抽样不是一个精确的操作）来抽样某些buckects。例如，下面的查询返回半数的buckets：

```sql
hive> SELECT * FROM bucketed_users
    > TABLESAMPLE(BUCKET 1 OUT OF 2 ON id);
4    Ann
0    Nat
2    Joe
```

抽样一个bucketed表是非常高效，因为查询只读取匹配**TABLESAMPLE**子句的buckets。与使用_rand\(\)_函数抽样一个nonbucketed表进行相比，后者会扫描整个输入数据集，即使是只需要一个很小的样本：

```
hive> SELECT * FROM users
    > TABLESAMPLE(BUCKET 1 OUT OF 4 ON rand());
2    Joe
```

### 6.3、Storage Formats

Hive中控制表存储的两个维度：**行格式**（row format）和**文件格式**（file format）。行格式决定行和某行的属性的存储。用Hive术语来说，行格式由一个 _**SerDe**_（一个序列化器和反序列化器的复合词）定义。

查询表时，SerDe 表现为一个反序列化器，它会将文件中一行字节数据反序列化为Hive内部使用的对象（Hive对这个行数据对象执行操作）。当执行 **INSERT** 或者 **CTAS**（查看[导入数据]()章节）时，SerDe 作为序列化器，表的SerDe 会将一行数据的 Hive 内部表示序列化为写到输出文件的字节。

文件格式决定了一个行中属性的容器格式（container format for fields in a row）。最简单的格式是普通文本文件，但是也有面向行（row-oriented）和面向列（column-oriented）二进制格式可以使用。

#### 6.3.1、The Default storage format：Delimited text

如果创建Hive表时没有使用 **ROW FORMAT** 和 **STORED AS** 子句，默认格式是一行一条的定界文本（delimited text with one row per line）。

默认的**行定界符**（row delimiter，分隔属性）不是制表符，而是 ASCII 控制字符集中的 Ctrl-A 符（ASCII 码是1）。选择使用Ctrl-A，在文档中有时写作^A，是因为相比制表符，在文本中出现 ^A 的概率更少。在 Hive 中没有转义定界符的方法，所以选择不会在数据属性中出现的字符很重要。

默认的**集合定界符**（collection item delimiter）是 Ctrl-B，用于给 ARRAY、STRUCT、或者 Map 的键值对中的条目定界。默认的map键定界符是Ctrl-C，用于界定 Map 中的键和值。

表中的**行则使用换行符进行定界**。

**警告**：之前关于定界符的描述对于通常的扁平的数据结构，只包含基本类型的复杂类型是正确的。但是，对于嵌套类型，就没那么简单了，事实上嵌套的层级决定了定界符。例如，对于一个数组的数组，外层数组定界符是Ctrl-B，但是内层数组定界符是Ctrl-C——ASCII控制符集中的下一个。Hive 事实上支持 8 层级别的定界符，对应于ASCII码的1-8，但是只能重写（override，重新定义，重置）前三个。

综上所述，建表语句：

```SQL
CREATE TABLE ...;
```

等价于更明确的语句：

```sql
CREATE TABLE ...
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\001'
  COLLECTION ITEMS TERMINATED BY '\002'
  MAP KEYS TERMINATED BY '\003'
  LINES TERMINATED BY '\n'
STORED AS TEXTFILE;
```

注意，定界符的八进制格式：例如 \001 代表 Ctrl-A。

Hive内部使用一个叫作 _LazySimpleSerDe_ 的 SerDe 用于这种定界的格式（即Delimited text），与面向行的MapReduce文本输入和输出格式一起使用。“lazy”前缀表示延迟地反序列化属性——只在被访问的时候。但是，这不是一个紧凑的格式，因为属性是以冗余文本格式（verbose textual format）保存的，例如，Boolean 值保存为字面的字符串 _true_ 或者 _false_。

Delimited text 这种格式很简单，但是，相比之下，二进制存储格式更加的紧凑和高效。

#### 6.3.2、Binary storage formats：Sequence files，Avro datafiles，Parquet files，RCFiles，and ORCFiles

使用二进制存储格式简单到只要更改建表语句CREATE TABLE的**STORED AS**子句即可。这种情况下，不用指定**ROW FORMAT**子句，因为行格式由底层的二进制文件格式控制。

二进制格式可以被划分为两种类型：面向行的格式和面向列的格式。一般来讲，面向列的格式在只查询表的一小部分字段时性能更好，而面向行的格式适用于需要同时处理一行的很多列数据的情况。

Hive自身支持的两种面向行的格式是Avro datafiles和顺序文件。两者都是通用的、可切分的、可压缩的格式；另外，Avro支持schema演变和多种语言绑定。在Hive 0.14.0中，可以用Avro格式保存表：

```
SET hive.exec.compress.output=true;
SET avro.output.codec=snappy;
CREATE TABLE ... STORED AS AVRO;
```

注意，通过设置相关的属性，为表开启了压缩。

类似地，通过声明**STORED AS SEQUENCEFILE**可以用于把表保存为顺序文件。

Hive也支持Parquet、RCFile和ORCFile这些面向列的二进制格式。

```sql
CREATE TABLE users_parquet STORED AS PARQUET
AS
SELECT * FROM users;
```

#### 6.3.3、Using a custom SerDe：RegexSerDe

使用自定义的SerDe来加载数据。用一个使用正则表达式的SerDe从一个文本文件读取定宽站点元数据：

```sql
CREATE TABLE stations (usaf STRING, wban STRING, name STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  "input.regex" = "(\\d{6}) (\\d{5}) (.{29}) .*"
);
```

在之前的例子里，在**ROW FORMAT**子句中使用了**DELIMITED**关键字指向了delimited text格式。在这个例子里，使用**SERDE**关键字和实现了这个SerDe的Java类的全限定类名（_org.apache.hadoop.hive.contrib.serde2.RegexSerDe_），指定了一个SerDe。

可以使用**WITH SERDEPROPERTIES**子句用额外的属性来配置SerDe。这里设置了_input.regex_属性，这是_RegexSerDe_的专用属性。

可以用**LOAD DATA**语句为这个表加载数据：

```sql
LOAD DATA LOCAL INPATH "input/ncdc/metadata/stations-fixed-width.txt"
INTO TABLE stations;
```

在执行加载时是没有使用表的SerDe的。当从表获取数据时，会调用表的SerDe来进行反序列化，如下，正确地解析了每行的属性：

```
hive> SELECT * FROM stations LIMIT 4;
010000    99999    BOGUS NORWAY                 
010003    99999    BOGUS NORWAY                 
010010    99999    JAN MAYEN                    
010013    99999    ROST
```

_RegexSerDe_比较低效，不通用。对于一般表应该优先考虑二进制存储格式。

#### 6.3.4、Storage handlers

Storage handlers用于Hive不能自然地访问的存储系统，比如HBase。使用**STORED BY**子句来指定storage handlers。

### 6.4、Importing Data

使用**LOAD DATA**语句，可以通过复制或者移动文件到表目录的方式把数据导入到Hive表。也可以通过使用INSERT语句或者_CTAS_（CREATE TABLE ... AS SELECT的缩写）结构，把数据从一个Hive表加载到另一个Hive表中。

如果要从关系型数据库直接把数据导入Hive，考虑Sqoop；详见[Imported Data and Hive]()章节。

#### 6.4.1、Inserts

**INSERT**语句：

```sql
INSERT OVERWRITE TABLE target
SELECT col1, col2
  FROM source;
```

对于分区表，要通过增加**PARTITION**子句来指定要插入的分区：

```sql
INSERT OVERWRITE TABLE target
PARTITION (dt='2001-01-01')
SELECT col1, col2
  FROM source;
```

**OVERWRITE**关键字意味着，表_target_（第一个例自）或者分区_2001-01-01_（第二个例子）会被**SELECT**语句的结果替换。如果要往未分区的表或者分区增加数据，那么使用**INSERT INTO TABLE**。

```sql
--非分区表，按默认字段顺序插入
INSERT INTO TABLE target values(val1,val2,val3,...);
--非分区表，指定字段顺序插入
INSERT INTO TABLE target (col1,col3,col2) values(val1,val3,val2,...);
--分区表，分区必须使用PARTITION关键字指定，不能放在普通字段
INSERT INTO TABLE target PARTITION(pt='pt1') values(val1,val2,val3,...);
INSERT INTO TABLE target PARTITION(pt='pt1') (col1,col3,col2) values(val1,val3,val2,...);
```

以上是静态分区插入，动态分区插入：

```sql
--分区表，动态分区
--首先必须开启动态分区，否则如下的插入语句会报错
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
INSERT INTO TABLE target PARTITION(pt) values(val1,val2,val3,...,'pt1');
```

通过从**SELECT**语句决定分区值，可以动态的指定分区（即动态分区插入——_dynamic partition insert_）：

```sql
INSERT OVERWRITE TABLE target
PARTITION (dt)
SELECT col1, col2, dt
  FROM source;
```

#### 6.4.2、Multitable insert

在HiveQL中，可以**INSERT**语句可以有如下的变换，效果一样：

```sql
FROM source
INSERT OVERWRITE TABLE target
  SELECT col1, col2;
```

这种语法使一个查询语句支持多个**INSERT**语句。_multitable insert_比多个**INSERT**语句更加高效，因为只用扫描一次源表便可以产生出不相交的多个输出。例如：

```sql
FROM records2
INSERT OVERWRITE TABLE stations_by_year
  SELECT year, COUNT(DISTINCT station)
  GROUP BY year 
INSERT OVERWRITE TABLE records_by_year
  SELECT year, COUNT(1)
  GROUP BY year
INSERT OVERWRITE TABLE good_records_by_year
  SELECT year, COUNT(1)
  WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
  GROUP BY year;
```

只有一个源表，但是有三个表来保存来自三个不同查询语句的结果。

#### 6.4.3、CREATE TABLE ... AS SELECT

把Hive查询的输出保存在一个新的表中通常是很方便的，可能的原因是查询结果太大而不能导出到console，或者因为需要在查询结果上进行更多的处理。

新表的列定义派生自**SELECT**子句获取的列。在如下查询中，表_target_有两列_col1_和_col2_，它们的类型和_source_表中的对应列相同：

```sql
CREATE TABLE traget
AS
SELECT col1, col2, dt
  FROM source;
```

CTAS操作是原子的，所以，如果**SELECT**查询因为某些原因失败，将不会创建表。

### 6.5、Altering Tables

因为Hive使用schema-read的方式，可以很灵活的在表创建之后修改表的定义。一般需要注意的是，要确保数据的改变能够反映表的新结构。

重命名表：

```sql
ALTER TABLE source RENAME TO target;
```

除了修改了表的元数据以外，对于内部表，**ALTER TABLE**语句把底层的表目录进行了移动来对应新的表名。这个例子里，_/user/hive/warehouse/source_ 被重命名为 _/user/hive/warehouse/target_。但是，对于外部表，只会修改表的元数据，不会移动底层目录。

也可以修改Hive表的列定义，增加新列，甚至可以替换表中的所有既存列。例如，增加新列：

```sql
ALTER TABLE target ADD COLUMNS (col3 STRING);
```

新列被加在（没有分区的）既存列后面。不会更新数据文件，所以对于_col3_的查询都是返回_null_（除非，数据文件中已经存在了额外的数据区域）。因为Hive不允许更新数据，需要通过其它的机制更改底层的文件。因此，更常见的操作是创建新的Hive表而不是修改Hive表定义。

**※** 对于分区表，如果对表增加了字段后，默认只会修改表的元数据；表中既存分区的分区元数据则不会发生改变，对旧的分区进行INSERT操作时，会报错。要避免这种情况，增加字段时，需要用到**CASCADE**关键字（CASCADE只适用于分区表？）。CASCADE对于改变字段顺序的改变也是无能为力的，即使使用CASCADE老的分区因为字段更改造成的字段错位仍让存在。总之，只要更改字段顺序，旧分区的数据都会错位，不同的是，加了CASCADE后对旧分区的插入不会报错。

```sql
ALTER TABLE target ADD COLUMNS( col3 STRING) CASCADE;
```

不加**CASCADE**时使用的是默认的**RESTRICT**，只会更新表的元数据，而不更新分区元数据；而使用了**CASCADE**关键字会更新表元数据和所有分区的元数据。

添加分区：

```sql
ALTER TABLE target ADD IF NOT EXISTS PARTITION(s_key='123');
```

**※** 没有分区的表，是不能增加分区的。想要使表改为分区表，大概要重建表吧？毕竟不是什么复杂的事。

**※** 另外，还有 metaStore 检查命令 **MSCK**（Hive MetaStore Consistency Check），可以用来添加 HDFS 实际存在，而 Hive 分区元数据中没有的分区；本分版本的 Hive （2.4.0，3.0.0，3.1.0）支持删除 HDFS 中不存在而 Hive Metastore 中还存在的分区。

```sql
MSCK REPAIR TABLE table_name
```

删除分区：

```sql
ALTER TABLE target DROP IF EXISTS PARTITION(s_key='123')
```

**外部表和内部表的转换**

```sql
--内部表转为外部表
ALTER TABLE table_name SET TBLPROPERTIES('EXTERNAL'='TRUE');

--外部表转为内部表
ALTER TABLE table_name SET TBLPROPERTIES('EXTERNAL'='FALSE');
```

### 6.6、Dropping Tables

**DROP TABLE**语句删除内部表的元数据和数据。对于外部表，只删除元数据；不会影响数据。

如果要删除表的数据，而只保留表的定义，可以使用**TRUNCATE TABLE**语句：

```sql
TRUNCATE TABLE my_table;
```

对于外部表不起作用，可以使用_dfs -rmr_（从Hive shell中）直接删除外部表的目录。

可以使用**LIKE**关键字创建一个与既存表相同schema的空表：

```sql
CREATE TABLE new_table LIKE existing_table;
```
