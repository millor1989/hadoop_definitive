# Chapter 17. Hive

Apache Hive是一个基于Hadoop的用于数据建仓的（data warehousing）框架。Hive最初被创建的目的是，给拥有较强SQL技能而Java编程能力较弱的分析师，用于查询保存在HDFS中的大量数据。如今，已经是一个通过的，可扩展的数据处理平台。

当然，SQL并不是对所有大数据问题都适用——例如，它不适用与构建复杂的机器学习算法——但是对许多分析很有用，它的优势是拥有广泛的用户基础。另外，SQL是商业智能（business intelligence）工具（比如，ODBC）中的通用语，所以Hive可以很好的和这些产品集成。

Hive本身不提供数据存储功能，它使用HDFS存储数据；

Hive不是分布式计算框架，核心是将SQL语句转换为（MapReduce）任务执行；

Hive不是资源调度系统，默认由Hadoop中YARN调度资源。