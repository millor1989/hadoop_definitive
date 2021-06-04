## 5、Java接口（The Java Interface）

这一节深入Hadoop _**FileSystem**_这个类：和Hadoop文件系统交互的API。尽管关注的主要是HDFS实现，_**DistributedFileSystem**_，通常，应该努力使用_**FileSystem**_这个抽象类来写代码，以保留跨文件系统的可能性。这在测试程序的时候是很有用的，例如，可以使用本地文件系统保存的数据快速地进行测试。



