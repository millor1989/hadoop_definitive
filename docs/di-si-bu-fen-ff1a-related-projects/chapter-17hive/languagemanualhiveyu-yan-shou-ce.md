## 1.Select

### 1.1、Select语法

```sql
[WITH CommonTableExpression (, CommonTableExpression)*]    (Note: Only available starting with Hive 0.13.0)
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference
  [WHERE where_condition]
  [GROUP BY col_list]
  [ORDER BY col_list]
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
  ]
 [LIMIT [offset,] rows]
```

- SELECT语句可以是UNION查询的一部分，也可以是另一个查询的子查询。

- *table_reference*指的是查询的输入。可以是一个普通的表、一个视图、一个join construct或者一个子查询。

- 表名和列名是大小写不敏感的

- 简单查询。例如，如下查询获取表1的所有记录。

  ```sql
  SELECT * FROM t1
  ```

  注意：对于Hive 0.13.0，**FROM**子句是可选的（例如，SELECT 1+1）。

- 使用current_database()函数获取当前的数据库（Hive 0.13.0）

  ```sql
  SELECT current_database()
  ```

- 要指定数据库，可以用数据库名限定表名（“*db_name.table_name*”，从Hive 0.7开始），也可以在查询语句之前使用**USE**语句（自Hive 0.6开始）。

  使用“*db_name.table_name*”，可以访问不同数据库中的表。

  **USE**设置所有后续HiveQL语句的数据库。USE加“*default*”关键字，重设数据库为默认数据库。

  ```sql
  USE database_name;
  SELECT query_specifications;
  USE default;
  ```

#### 1.1.1、WHERE Clause

**WHERE**条件是一个boolean表达式。Hive的**WHERE**子句支持许多操作符和UDFs。

```sql
SELECT * FROM sales WHERE amount > 10 AND region = "US"
```

对于Hive 0.13，**WHERE**子句不支持某些类型的子查询。

#### 1.1.2、ALL and DISTINCT Clauses

**ALL**和**DISTINCT**选项指定是否返回重复的行。如果不指定，默认的是**ALL**（返回所有的匹配行）。指定**DISTINCT**会从结果集中移除重复的行。注意，从1.1.0版本开始，Hive支持***SELECT DISTINCT \****。

```sql
hive> SELECT col1, col2 FROM t1
    1 3
    1 3
    1 4
    2 5
hive> SELECT DISTINCT col1, col2 FROM t1
    1 3
    1 4
    2 5
hive> SELECT DISTINCT col1 FROM t1
    1
    2
```

**ALL**和**DISTINCT**都可以用于**UNION**子句。

#### 1.1.3、Partition Based Queries

通常，SELECT查询扫描整个表（去了抽样sampling）。如果表是使用**PARTITIONED BY**子句创建的，查询可以进行分区精简（partition pruning）并且只扫描查询指定的那些相关分区。目前，如果在**WHERE**子句或者**JOIN**的**ON**子句中指定了分区，Hive会进行分区精简。例如，对于分区表*page_views*，分区字段*date*，如下查询，只扫描日期在*2008-03-01*和*2008-03-31*之间的数据。

```sql
SELECT page_views.*
FROM page_views
WHERE page_views.date >= '2008-03-01' AND page_views.date <= '2008-03-31'
```

如果表*page_views*和另一个表*dim_users*进行连接，可像如下那样在**ON**子句中指定分区范围：

```sql
SELECT page_views.*
FROM page_views JOIN dim_users
  ON (page_views.user_id = dim_users.id AND page_views.date >= '2008-03-01' AND page_views.date <= '2008-03-31')
```

##### 1.1.3.1、Partition Filter Syntax

对于有分区键*country*和*state*的表，可以构建如下过滤器：

```sql
country = "USA" AND (state = "CA" OR state = "AZ")
```

特别注意：在括号中可以嵌入子表达式。

可以使用如下操作符来构建分区字段过滤器：

- =
- &lt;
- &lt;=
- &gt;
- &gt;=
- &lt;&gt;
- AND
- OR
- LIKE（只对*string*类型键起作用，支持带有'.*'通配符的文本字符串模板）

#### 1.1.4、HAVING Clause

Hive在0.7.0版本增加了对HAVING子句的支持。在更老的Hive版本中可以通过子查询达到相同的效果。

```sql
SELECT col1 FROM t1 GROUP BY col1 HAVING SUM(col2) > 10
```

等效子查询：

```sql
SELECT col1 FROM (SELECT col1, SUM(col2) AS col2sum FROM t1 GROUP BY col1) t2 WHERE t2.col2sum > 10
```

#### 1.1.5、LIMIT Clause

LIMIT子句可以用来限制SELECT语句返回的行数。可以使用一个或者两个数值参数，必须都是正整数。第一个参数用来指定要返回的第一行的偏移量，第二个参数指定返回的最大行数。只有一个参数时，这个参数表示返回结果的最大行数，而第一行的偏移量为默认值0。

如下查询返回任意5个costomers：

```sql
SELECT * FROM customers LIMIT 5
```

如下查询返回首先创建的5个customers：

```sql
SELECT * FROM customers ORDER BY create_date LIMIT 5
```

如下查询返回创建的第三个和第七个customers：

```sql
SELECT * FROM customers ORDER BY create_date LIMIT 2,5
```

#### 1.1.6、REGEX Column Specification

在Hive 0.13.0之前的版本中**SELECT**语句可以使用基于正则表达式的列指定，在Hive 0.13.0和之后的版本中，可以通过设置配置属性`hive.support.quoted.identifiers`为*none*来开启这个功能 。

## 2、Group By

### 2.1、Group By Syntax

```sql
groupByClause: GROUP BY groupByExpression (, groupByExpression)*
 
groupByExpression: expression
 
groupByQuery: SELECT expression (, expression)* FROM src groupByClause?
```

在*groupByExpression*中，通过字段名称而不是位置指定字段。但是，在Hive 0.11.0及之后的版本中，通过如下配置可以实现按照位置指定字段：

- 对于Hive 0.11.0到2.1.x，设置`hive.groupby.orderby.position.alias` 属性为*true*（默认*false*）
- 对于Hive 2.2.0和以后的版本，设置`hive.groupby.position.alias`为*true*（默认为*false*）

**例子**

查询表中数据的行数：

```sql
SELECT COUNT(*) FROM table2;
```

按照gender查询不同users的数量：

```sql
INSERT OVERWRITE TABLE pv_gender_sum
SELECT pv_users.gender, count (DISTINCT pv_users.userid)
FROM pv_users
GROUP BY pv_users.gender;
```

可以同时进行多个聚合，但是，不能同时对两个不同字段的**DISTINCT**进行聚合。例如，如下查询可以是因为**count(DISTINCT)**和**sum(DISTINCT)**是针对同一个字段的：

?????为什么HUE Hive和Spark中都支持多个不同字段的DISTINCT???HUE Impala中不支持不同字段DISTINCT聚合??

```sql
INSERT OVERWRITE TABLE pv_gender_agg
SELECT pv_users.gender, count(DISTINCT pv_users.userid), count(*), sum(DISTINCT pv_users.userid)
FROM pv_users
GROUP BY pv_users.gender;
```

但是，如下的查询是不可以的，一个查询中不能有多个**DISTINCT**表达式：

```sql
INSERT OVERWRITE TABLE pv_gender_agg
SELECT pv_users.gender, count(DISTINCT pv_users.userid), count(DISTINCT pv_users.ip)
FROM pv_users
GROUP BY pv_users.gender;
```

#### 2.1.2、Select statement and group by clause

当时使用了*group by*子句时，*select*语句只能包括*group by*子句中的字段。当然，可以有*select*语句中可以有多个聚合函数。

举个例子：

```sql
CREATE TABLE t1(a INTEGER, b INTGER);
```

上表的一个group by查询：

```sql
SELECT
   a,
   sum(b)
FROM
   t1
GROUP BY
   a;
```

下面的group by查询是不合语法的：

```sql
SELECT
   a,
   b
FROM
   t1
GROUP BY
   a;
```

### 2.2、Advanced Features

#### 2.2.1、Multi-Group-By Inserts

聚合或者简单select的输出可以插入多个表甚至是hadoop dfs文件（可以使用hdfs工具进行操作）中。如果既要按照*gender*统计uv，也要按照*age*统计uv，可以使用如下查询：

```sql
FROM pv_users
INSERT OVERWRITE TABLE pv_gender_sum
  SELECT pv_users.gender, count(DISTINCT pv_users.userid)
  GROUP BY pv_users.gender
INSERT OVERWRITE DIRECTORY '/user/facebook/tmp/pv_age_sum'
  SELECT pv_users.age, count(DISTINCT pv_users.userid)
  GROUP BY pv_users.age;
```

#### 2.2.2、Map-side Aggregation for Group By

`hive.map.aggr`控制如何进行聚合。默认值是*false*。如果设置为*true*，Hive会在map task中执行first-level的聚合。这通常效率很高，但是需要更多的内存才能成功执行。

```sql
set hive.map.aggr=true;
SELECT COUNT(*) FROM table2;
```

#### 2.2.3、Enhanced Aggregation, Cube, Grouping and Rollup

[略了吧](https://cwiki.apache.org/confluence/display/Hive/Enhanced+Aggregation%2C+Cube%2C+Grouping+and+Rollup)

## 3、SortBy

### 3.1、Syntax of Order By

Hive QL的*ORDER BY*语法与SQL语言的*ORDER BY*语法类似：

```sql
colOrder: ( ASC | DESC )
colNullOrder: (NULLS FIRST | NULLS LAST)           -- (Note: Available in Hive 2.1.0 and later)
orderBy: ORDER BY colName colOrder? colNullOrder? (',' colName colOrder? colNullOrder?)*
query: SELECT expression (',' expression)* FROM src orderBy
```

在“order by”子句中有些限制：在严格模式下（`hive.mapred.mode`=strict），order by子句必须跟着“limit”子句。如果设置`hive.mapred.mode`为nonstrict，limit子句则是不必需要加的。原因是，为了给所有结果强加一个整体的顺序，需要有一个reduce对最终输出进行排序。如果输出行数太大，那么仅有的一个reducer需要耗时很长才会完成。

注意，要通过字段名称而不是位置指定字段。但是，在Hive 0.11.0及之后的版本中，通过如下配置可以实现按照位置指定字段：

- 对于Hive 0.11.0到2.1.x，设置`hive.groupby.orderby.position.alias` 属性为*true*（默认*false*）
- 对于Hive 2.2.0和以后的版本，设置`hive.groupby.position.alias`为*true*（默认为*false*）

默认的排序顺序是升序（ASC）。

在Hive 2.1.0和之后的版本中，支持在“order by”子句中为每个排序字段指定null的排序顺序。ASC的默认null排序顺序是*NULLS FIRST*，而DESC的默认null排序顺序是*NULLS LAST*。

在Hive 3.0.0和之后的版本中，视图和子查询中没有limit的order by会被优化器移除。设置`hive.remove.orderby.in.subquery`为false，可以这个优化。

### 3.2、Syntax of Sort By

*SORT BY*语法和SQL语言中的*ORDER BY*语法类似。

```sql
colOrder: ( ASC | DESC )
sortBy: SORT BY colName colOrder? (',' colName colOrder?)*
query: SELECT expression (',' expression)* FROM src sortBy
```

Hive使用*SORT BY*中的字段，在把记录给reducer之前，对记录进行排序。排序顺序依字段类型而定。如果字段是数值类型，那么排序顺序也是数值顺序。如果字段是字符串类型的，那么排序属性是字典顺序。

在Hive 3.0.0和之后的版本中，视图和子查询中没有limit的sort by会被优化器移除。设置`hive.remove.orderby.in.subquery`为false，可以这个优化。

#### 3.2.1、Difference between Sort By and Order By

Hive支持*SORT BY*，按照每个reducer进行排序。“order by”和“sort by”的不同是，前者为整个输出结果排序，而后者只对同一个reducer中的记录进行排序。如果有超过一个的reducer，“sort by”可能会产生部分排序的最终输出结果。

注意：针对一个字段的SORT BY和CLUSTER BY的区别，如果有多个reducers随机地分区，CLUSTER BY按照field分区并且SORT BY，以便在reducers之间均匀地分布数据（和负载）。

基本上，每个reducer中的数据会根据用户指定的顺序排序。如下例子：

```sql
SELECT key, value FROM src SORT BY key ASC, value DESC
```

这个查询有两个reducers，输出分别是：

```
0   5
0   3
3   6
9   1
```

```
0   4
0   3
1   1
2   5
```

#### 3.2.2、Setting Types for Sort By

转换（TRANSFORM）之后的变量类型通常被认为是字符串，这意味着数值数据会按照字典顺序排序。要避免这种情况，在使用SORT BY之前，可以增加一个带有强转的SELECT语句。

```sql
FROM (FROM (FROM src
            SELECT TRANSFORM(value)
            USING 'mapper'
            AS value, count) mapped
      SELECT cast(value as double) AS value, cast(count as int) AS count
      SORT BY value, count) sorted
SELECT TRANSFORM(value, count)
USING 'reducer'
AS whatever
```

### 3.3、Syntax of Cluster By and Distribute By

*Cluster By*和*Distribute By*主要和Transform/Map-Reduce脚本一起使用。但是，它有时在SELECT语句中有用，如果为了后续的查询，要对一个查询的输出进行分区和排序。

*Cluster By*是*Distribute By*和*Sort By*的简称。

Hive使用*Distribute By*中的字段来在reducers之间分布记录。*Distribute By*字段相同的记录会在相同的reducer中。但是，*Distribute By*不保证基于distributed keys进行聚合或者排序。

例如，将如下5行记录*Distributing By*到两个reducers：

```
x1
x2
x4
x3
x1
```

reducer 1对应结果为：

```
x1
x2
x1
```

reducer 2对应结果为：

```
x4
x3
```

注意，可以保证用于相同键*x1*的记录分布在相同的reducer中，但是不保证相同键的记录位置上相邻。

相反的，如果使用*Cluster By x*，两个reducers会进一步基于*x*进行排序：

reducer 1对应结果为：

```
x1
x1
x2
```

reducer 2对应结果为:

```
x3
x4
```

除了指定*Cluster By*，还可以同时指定*Distribute By*和*Sort By*，那么分区字段和排序字段是不同的。通常情况是，分区字段是排序字段的前置，但是这不是强制的。

```sql
SELECT col1, col2 FROM t1 CLUSTER BY col1

SELECT col1, col2 FROM t1 DISTRIBUTE BY col1
 
SELECT col1, col2 FROM t1 DISTRIBUTE BY col1 SORT BY col1 ASC, col2 DESC
```

```sql
FROM (
  FROM pv_users
  MAP ( pv_users.userid, pv_users.date )
  USING 'map_script'
  AS c1, c2, c3
  DISTRIBUTE BY c2
  SORT BY c2, c1) map_output
INSERT OVERWRITE TABLE pv_users_reduced
  REDUCE ( map_output.c1, map_output.c2, map_output.c3 )
  USING 'reduce_script'
  AS date, count;
```

## 4.Hive Joins

### 4.1、Join Syntax

Hive支持如下的表连接语法：

```sql
join_table:
    table_reference [INNER] JOIN table_factor [join_condition]
  | table_reference {LEFT|RIGHT|FULL} [OUTER] JOIN table_reference join_condition
  | table_reference LEFT SEMI JOIN table_reference join_condition
  | table_reference CROSS JOIN table_reference [join_condition] (as of Hive 0.10)

table_reference:
    table_factor
  | join_table

table_factor:
    tbl_name [alias]
  | table_subquery alias
  | ( table_references )

join_condition:
    ON expression
```

从Hive 0.13.0开始支持隐式的连接符号（implicit join notation）。这允许在FROM子句中逗号分隔的表集合之间进行连接，而不需要JOIN关键字。例如：

```sql
SELECT *
FROM table1 t1, table2 t2, table3 t3
WHERE t1.id = t2.id AND t2.id = t3.id AND t1.zipcode = '02535';
```

从Hive 0.13.0开始支持非限定列引用（unqualified column references）。如果非限定列引用指向多个表，Hive会把它标记为ambiguous引用。例如：

```sql
CREATE TABLE a (k1 string, v1 string);
CREATE TABLE b (k2 string, v2 string);

SELECT k1, v1, k2, v2
FROM a JOIN b ON k1 = k2; 
```

从Hive 2.2.0开始支持ON子句中的复杂连接表达式（complex join expression）。但是，优先级更高的是，Hive不支持非等值的连接条件。

连接条件语法限制如下：

```
join_condition:
    ON equality_expression ( AND equality_expression )*
equality_expression:
    expression = expression
```

### 4.2、Examples

写join查询时要考虑的一些显著的点：

- 支持复杂连接表达式

  ```sql
   SELECT a.* FROM a JOIN b ON (a.id = b.id);
   SELECT a.* FROM a JOIN b ON (a.id = b.id AND a.department = b.department);
   --不是不支持非等值的连接条件吗？？？
   SELECT a.* FROM a LEFT OUTER JOIN b ON (a.id <> b.id);
  ```

- 一个查询中可以有超过两个的表进行连接

  ```sql
   SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
  ```

- 对于多个表的连接，如果join子句中使用的是相同的字段，Hive会把它转换为同一个map/reduce job。对于：

  ```sql
  SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
  ```

  Hive会把它转换为同一个map/reduce job，因为只有join中只涉及了b表的*key1*字段。而：

  ```sql
  SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
  ```

  会被Hive转换为两个map/reduce job，因为第一个join用到了表b的*key1*并且第二个join用到了表b的*key2*字段。第一个map/reduce执行a表和b表的连接，结果再和c表在第二个map/reduce中进行连接。

- 在join的map/reduce的每个stage，join序列中的最后一个表在reducers之间流动，而其它表会被缓存。因此，把最大的表放在join序列的最后，能够减少reducer中内存的使用。例如，查询：

  ```sql
  SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
  ```

  中三个表在同一个map/reduce job中进行连接，表a和表b中对应key的某些特定值缓存在reducers的内存中。然后，从表c中取出的每行，与缓存的行进行连接。类似地：

  ```sql
  SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
  ```

  查询的第一个map/reduce缓存表a，第二个map/reduce缓存第一个map/reduce的结果。

- 在join的map/reduce的每个stage，流动的表可以通过暗示指定。在查询：

  ```sql
  SELECT /*+ STREAMTABLE(a) */ a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
  ```

  中，流动的表是a表。如果没有**STREAMTABLE**暗示，Hive会流动join中最右的表。

- **LEFT**, **RIGHT**, 和**FULL OUTER**连接的存在是为了为ON子句中不匹配的记录进行更多的控制。例如，查询：

  ```sql
  SELECT a.val, b.val FROM a LEFT OUTER JOIN b ON (a.key=b.key)
  ```

  会返回a表中的每一行记录。如果ON子句匹配，结果是(a.val,b.val)，否则是(a.val,NULL)。逻辑与基本的SQL连接一样。

- 连接中的ON子句比WHERE子句先执行。WHERE子句过滤条件可以考虑放在ON子句中。

- 连接顺序是不可交换的。多表连接，改变顺序会影响结果。