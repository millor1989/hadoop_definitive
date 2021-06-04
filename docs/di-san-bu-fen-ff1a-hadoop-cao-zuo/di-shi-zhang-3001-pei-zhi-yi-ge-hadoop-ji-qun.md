# Chapter 10.Setting Up a Hadoop Cluster

本章介绍如何配置一个Hadoop在机器集群上运行。运行HDFS、MapReduce、YARN于一个集群对学习这些系统有用，但是要做有用的工作，这些系统需要运行在多个节点上。

安装选则：

- Apache tarballs

  Apache Hadoop工程和相关的工程都提供了每个版本的二进制（和源码）tarballs。使用二进制tarballs安装能够带来最大的灵活性但是也需要相当大工作量，因为需要决定在文件系统哪里放置安装文件、配置文件、日志文件，还要正确配置它们的文件访问权限，等等。

- Packages

  [Apache BigTop project](http://bigtop.apache.org/)和所有的Hadoop发行者都有可用的RPM和Debian包。Packages与tarballs相比有许多好处：提供了一致的文件布局；作为一体进行了测试；能够与配置管理工具（Puppet）一起工作。

- Hadoop cluster management tools

  Cloudera管理器和Apache Ambari是安装和管理Hadoop集群的专用工具。它们提供一个简单的web UI，对大多数用户和操作者来说这是配置Hadoop集群的推荐方式。这些工具包含了运行Hadoop的许多操作知识。例如，它们使用启发式的硬件配置以为Hadoop配置设置选择好的默认值。对于更复杂的设置，如HA，或者安全Hadoop，管理工具提供充分测试过的向导以在短时间内获得一个运行的集群。最后，它们提供其它安装选项不提供的额外特征，例如统一监控和日志检索，以及滚动升级（rolling upgrades，不用经历下线就能升级集群）。

