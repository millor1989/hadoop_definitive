## 7、Querying Data

### 7.1、Sorting and Aggregating

使用标准的**ORDER BY**子句可以对Hive中的数据进行排序。**ORDER BY**执行一个并行的全部输入数据的排序。当不需要全局的排序的时候——多数情况不需要——可以使用Hive的非标准扩展**SORT BY**代替。**SORT BY**为每个reducer生成一个排序的文件。

某些情况下，要控制特定行所属的reducer——一般要进行子聚合。这是Hive的**DISTRIBUTE BY**子句的功能。如下例子按照_year_和_temperature_排序天气数据，并且确保了某年的所有数据在相同的reducer分区中。

```
hive> FROM records2
    > SELECT year, temperature
    > DISTRIBUTE BY year
    > SORT BY year ASC, temperature DESC;
1949    111
1949    78
1950    22
1950    0
1950    -11
```

如果**SORTED BY**和**DISTRIBUTED BY**子句的列是相同的，可以使用**CLUSTER BY**作为指定两者的快捷方式。

### 7.2、MapReduce Scripts

使用与Hadoop Streaming类似的方式，**TRANSFORM**、**MAP**和**REDUCE**子句支持引用外部的脚本或者程序。

###### 例 17-1.Python script to filter out poor-quality weather records

```python
#!/usr/bin/env python
import re
import sys
for line in sys.stdin:
  (year, temp, q) = line.strip().split()
  if (temp != "9999" and re.match("[01459]", q)):
    print "%s\t%s" % (year, temp)
```

可以按照下面的方式使用上面的脚本：

```sql
hive> ADD FILE /Users/tom/book-workspace/hadoop-book/ch17-hive/src/main/python/is_good_quality.py;
hive> FROM records2
    > SELECT TRANSFORM(year, temperature, quality)
    > USING 'is_good_quality.py'
    > AS year, temperature;
1950    0
1950    22
1950    -11
1949    111
1949    78
```

在运行查询之前，要使用Hive注册这个脚本。因此，Hive知道文件发送到了Hadoop集群。

这个查询自身把year，temperature，quality属性作为制表符分隔的行以流的形式发送给脚本_is\_good\_quality.py_，并且把制表符分隔的输出解析为year，temperature属性来构成查询的输出。

这个例子没有reducers。如果用嵌套的形式（nested form）做这个查询，可以使用**Map**关键字和**Reduce**关键字指定一个map函数和一个reduce函数。

```sql
FROM (
  FROM records2
  MAP year, temperature, quality
  USING 'is_good_quality.py'
  AS year, temperature) map_output
REDUCE year, temperature
USING 'max_temperature_reduce.py'
AS year, temperature;
```

### 7.3、Joins

Hive只支持等值连接（equijoins）；支持多个JOIN...ON...子句。Hive会对连接进行优化，尽量使用进行连接的MapReduce jobs最少。一个join会被实现为一个MapReudce job，但是多个joins可能会被实现为少于joins数量的MapReduce jobs，如果连接条件都相同。使用**EXPLAIN**关键字可以查看查询使用的MapReduce jobs数量：

```sql
EXPLAIN
SELECT sales.*, things.*
FROM sales JOIN things ON (sales.id = things.id);
```

**EXPLAIN**的输出包含查询执行计划的很多详细信息，包括抽象语法树、Hive将执行的stages的依赖图、以及每个stage的信息。stages可能是MapReduce jobs或者文件移动操作。使用**EXPLAIN EXTEND**可以查看更多的详细信息。

目前Hive使用基于规则的（rule-based）查询优化器来决定如何执行查询，从Hive 0.14.0开始，可以选择使用基于成本的（cost-based）查询优化器。

#### 7.3.1、Inner joins

内连接：

```sql
SELECT sales.*, things.*
FROM sales JOIN things
ON (sales.id = things.id);
```

等价的写法是：

```sql
SELECT sales.*, things.*
FROM sales, things
WHERE sales.id = things.id;
```

#### 7.3.2、Outer joins

LEFT OUTER JOIN：

```
hive> SELECT sales.*, things.*
    > FROM sales LEFT OUTER JOIN things ON (sales.id = things.id);
Joe    2    2    Tie
Hank   4    4    Coat
Ali    0    NULL NULL
Eve    3    3    Hat
Hank   2    2    Tie
```

RIGHT OUTER JOIN：

```
hive> SELECT sales.*, things.*
    > FROM sales RIGHT OUTER JOIN things ON (sales.id = things.id);
Joe    2    2    Tie
Hank   2    2    Tie
Hank   4    4    Coat
Eve    3    3    Hat
NULL   NULL 1    Scarf
```

FULL OUTER JOIN：

```
hive> SELECT sales.*, things.*
    > FROM sales FULL OUTER JOIN things ON (sales.id = things.id);
Ali    0    NULL NULL
NULL   NULL 1    Scarf
Hank   2    2    Tie
Joe    2    2    Tie
Eve    3    3    Hat
Hank   4    4    Coat
```

#### 7.3.3、Semi joins

**IN**子查询：

```sql
SELECT *
FROM things
WHERE things.id IN (SELECT id from sales);
```

可以写为半连接：

```
hive> SELECT *
    > FROM things LEFT SEMI JOIN sales ON (sales.id = things.id);
2    Tie
4    Coat
3    Hat
```

**注意**：半连接的限制是，右表的列只能出现在**ON**子句中，不能出现在**SELECT**子句中。

Spark编程中支持对Hive进行反连接（left\_anti）操作，相当于NOT IN查询；但是不知道HiveQL是否支持（HUE中没有anti关键字）。

#### 7.3.4、Map joins

对于内连接：

```sql
SELECT sales.*, things.*
FROM sales JOIN things ON (sales.id = things.id);
```

如果一个表小到可以放进内存（比如things表），Hive可以把它加载到内存，从而可以在每个mappers中执行join。即map join。

执行这个查询的job没有reducers，所以这种查询对于**RIGHT**或者**FULL OUTER JOIN**不起作用，因为只有在所有输入的聚合（reduce）步骤才能检测到不匹配。

Map joins可以利用bucketed表的优势，因为，基于左表的一个bucket运行的的mapper只用加载右表的对应bucket来进行join。但是，需要通过配置来开启这个优化方案：

```
SET hive.optimize.bucketmapjoin=true;
```

### 7.4、Subqueries

Hive支持**SELECT**语句的**FROM**子句中嵌套子查询，和某些情况下**WHERE**子句中嵌套子查询。

```sql
SELECT station, year, AVG(max_temperature)
FROM (
  SELECT station, year, MAX(temperature) AS max_temperature
  FROM records2
  WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
  GROUP BY station, year
) mt
GROUP BY station, year;
```

**注意**：Hive允许不相关的子查询，即子查询是**WHERE**子句通过**IN**或者**EXISTS**引用的一个查询。Hive不支持相关的子查询，即子查询引用外部的查询。

### 7.5、Views

视图是通过**SELECT**语句定义的“虚拟表”（virtual table）。视图以不同于把数据保存在硬盘的方式向用户呈现数据。通常，视图的数据是表数据的简化或者表数据特定方式的聚合，使它便于进行进一步的处理。视图还可以用于限制用户访问它们被授权访问的表的子集。

在Hive中，创建视图时不会把它物化到硬盘；视图的**SELECT**语句在使用视图时才会执行。如果要基于基础表对视图执行大量的转换或者要频繁使用视图，最好创建一个保存视图内容的表（**CTAS**）。

创建视图：

```sql
CREATE VIEW valid_records
AS
SELECT *
FROM records2
WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9);
```

创建视图时，不会运行查询，只是简单的把它保存在metastore中。**SHOW TABLES**命令的输出包含视图；使用**DESCRIBE EXTENDED **_**view\_name**_命令可以查看视图的详细信息，包含定义视图使用的查询语句。

可以基于视图来创建视图：

```sql
CREATE VIEW max_temperatures (station, year, max_temperature)
AS
SELECT station, year, MAX(temperature)
FROM valid_records
GROUP BY station, year;
```

在这个视图定义中，明确的指定了列名。因为**SELECT**子句中用到了聚合函数，不指定列名的话Hive会自动指定一个列的别名。当然，也可以通过在SELECT子句中为聚合表达式指定别名来定义列名。

使用视图与使用子查询等价。Hive的视图是只读的。

  
**※** 默认创建的视图，可以看作是对Select语句的封装。如果使用包含了Impala不支持的函数（比如，lateral view）来创建视图，则Impala在使用这个视图的时候会报错（无法解析视图定义语句）。

#### MATERIALIZED VIEW（物化视图）

Hive从3.0.0开始支持Materialized View，物化的视图。创建完物化的视图后，它的内容会自动地通过执行创建语句中的查询进行装配。物化视图的创建语句是原子性的，只有当物化视图装配完查询结果数据后，物化视图才是对其它用户可见的。

**materialized view会随着源表的数据变化而更新吗？如果不更新，那么和CTAS又有什么区别呢？**

### 7.6、User-Defined Functions

略

