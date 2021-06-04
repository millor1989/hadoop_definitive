## 2、YARN Compared to MapReduce 1

最初版本Hadoop的MapReduce，被成为MapReduce 1以和Hadoop 2中使用YARN的MapReduce 2进行区分。

**Note.**MapReduce的新老API和MapReduce 1与MapReduce 2并不是一回事。MapReduce API指的是面向用户的客户端侧的，决定怎么写MapReduce程序，新老API是不相互兼容的。而MapReduce 1与MapReduce 2是运行MapReduce程序的不同方式。新或老API都能在MapReduce 1或MapReduce 2上运行。

MapReduce 1控制job执行进程的两种守护线程是：

- 一个***jobtracker***：通过调度task在***tasktrackers***上的运行协调在系统上运行的所有jobs，它保存着每个job的所有进度记录，如果task失败，***jobtracker***可以在不同的***tasktracker***上重新调度它。
- 一个或者多个***tasktrackers***：运行tasks并向***jobtracker***发送进度报告。

MapReduce 1中，**jobtracker**即要调度job（matching tasks and tasktrackers）也要监控task进度（追踪tasks，重启失败或慢的tasks，做task记录，例如维护counter totals）。在YARN中不同的实体处理这些职责：资源管理期和application master（每个MapReduce job一个）。**jobtracker**也负责保存完成的job的job历史记录，尽管可以运行一个隔离的job history server守护线程来降低**jobtracker**的负载；在YARN中对应的是timeline server，保存应用历史记录。与**tasktrackers**对应的是YARN的节点管理器。YARN的Container对应与MapReduce 1的*Slot*。

YARN是针对MapReduce 1的限制设计的。使用YARN的好处如下：

- **可扩展性（Scalability）**：限于**jobtracker**既要管理jobs又要管理tasks，MapReduce 1在4000节点和4000tasks时达到可扩展性的瓶颈。YARN通过划分资源管理器和application master的架构优势逾越了MapReduce 1的限制，可以扩展到10000节点和10000tasks。不同于**jobtracker**，YARN中的每一个应用实例（这里是MapReduce Job）都有一个专用的application master，运行于应用的生命周期；这种模型更接近于原始的Google MapReduce论文，它描述了如何启动一个master进程来协调运行在一组workers上的map和reduce tasks。
- **可用性（Availability）**：高可用性（HA）通常通过备份需要的状态实现，在服务守护线程失败时，另一个守护线程接管失败线程提供服务工作需要使用这些状态。可是**jobtracker**内存中大量快速变化的复杂状态使为**jobtracker**改造HA非常困难。在YARN中使用资源管理器和application master替代**jobtracker**，使变服务为高可用成为一个可以分而治之（divide-and-conquer）的问题：为资源管理器提供HA，然后为YARN应用提供HA（基于每个应用）。
- **利用率（Utilization）**：MapReduce 1中，每个**tasktracker**都陪配置使用一个静态分配的固定大小的“slots”，在配置时被划分为map slots和reduce slots。map slot只能运行 map task， reduce slot只能运行reduce task。在YARN中，节点管理器管理一个**资源池**，而不是固定数量的设定的slots；YARN上运行的MapReduce不会遇到reduce task必须等待因为集群上只有map slots可用的情况，MapReduce 1会遇到；在YARN上，如果有资源可以运行task，应用就可以使用这些资源。另外，YARN中的资源是**细粒度**的，应用可以请求他所需要的量的资源，而不是不可分割的（对与特定任务或大或小）slot。
- **多租户（Multitenancy）**：在某种程度上，YARN带来的最大的好处是它不仅仅对MapReduce，也对其它分布式应用敞开了Hadoop。MapReduce只是许多YARN应用中的一个。甚至可以允许用户在同一个YARN集群上运行不同版本的MapReduce，这让升级MapReduce的过程变得可控（Note.MapReduce的某些部分，例如job history server和shuffle handler，包括YARN本身，都需要在整个集群中升级）。

