## 2、YARN Compared to MapReduce 1

最初版本 Hadoop 的 MapReduce，被称为 MapReduce 1 以和 Hadoop 2 中使用 YARN 的 MapReduce 2 进行区分。

**Note.**  **MapReduce 的新老 API 和 MapReduce 1 与 MapReduce 2 并不是一回事**。MapReduce API 指的是面向用户的客户端侧的，决定怎么写 MapReduce 程序，新老 API 是不相互兼容的。而 MapReduce 1 与 MapReduce 2 是运行 MapReduce 程序的不同方式。新或老 API 都能在 MapReduce 1 或  MapReduce 2 上运行。

#### 2.1、MapReduce 1 和 MapReduce 2 比较

MapReduce 1 控制 job 执行进程的两种守护线程是：

- 一个 ***jobtracker***：通过调度 task 在 ***tasktrackers*** 上的运行协调在系统上运行的所有 jobs，它保存着每个 job 的所有进度记录，如果 task 失败，***jobtracker*** 可以在不同的 ***tasktracker*** 上重新调度它。
- 一个或者多个 ***tasktrackers***：运行 tasks 并向 ***jobtracker*** 发送进度报告。

MapReduce 1 中，**jobtracker** 既要调度 job（matching tasks and tasktrackers）也要监控 task 进度（追踪 tasks，重启失败或慢的 tasks，做 task 记录，例如维护 counter totals）。在 YARN 中不同的实体处理这些职责：资源管理器和 application master（每个MapReduce job一个）。**jobtracker** 也负责保存完成的 job 的 job 历史记录，尽管可以运行一个隔离的 job history server 守护线程来降低 **jobtracker** 的负载；在 YARN 中对应的是 timeline server，保存应用历史记录。与 **tasktrackers** 对应的是 YARN 的节点管理器（NodeManager）。YARN 的 Container 对应与 MapReduce 1 的 *Slot*。

YARN 是针对 MapReduce 1 的限制设计的。使用YARN的好处如下：

- **可扩展性（Scalability）**：限于 **jobtracker** 既要管理 jobs 又要管理 tasks，MapReduce 1 在 4000 节点和 4000 tasks 时达到可扩展性的瓶颈。YARN 通过划分资源管理器和 application master 的架构优势逾越了 MapReduce 1 的限制，可以扩展到 10000 节点和 10000 tasks。不同于 **jobtracker**，YARN 中的每一个应用实例（这里是MapReduce Job）都有一个专用的 application master，运行于应用的生命周期；这种模型更接近于原始的 Google MapReduce 论文，它描述了如何启动一个 master 进程来协调运行在一组 workers 上的 map 和 reduce tasks。
- **可用性（Availability）**：高可用性（HA）通常通过备份需要的状态实现，在服务守护线程失败时，另一个守护线程接管失败线程提供服务工作需要使用这些状态。可是 **jobtracker** 内存中大量快速变化的复杂状态使为 **jobtracker** 改造HA非常困难。在 YARN 中使用资源管理器和 application master 替代 **jobtracker**，使变服务为高可用成为一个可以分而治之（divide-and-conquer）的问题：为资源管理器提供 HA，然后为 YARN 应用提供 HA（基于每个应用）。
- **利用率（Utilization）**：MapReduce 1 中，每个 **tasktracker** 都配置使用一个静态分配的固定大小的 “slots”，在配置时被划分为 map slots 和 reduce slots。map slot 只能运行 map task， reduce slot 只能运行 reduce task。在 YARN 中，节点管理器管理一个**资源池**，而不是固定数量的设定的 slots；YARN 上运行的 MapReduce 不会遇到 reduce task 必须等待因为集群上只有 map slots 可用的情况，MapReduce 1 会遇到；在 YARN 上，如果有资源可以运行 task，应用就可以使用这些资源。另外，YARN 中的资源是**细粒度**的，应用可以请求他所需要的量的资源，而不是不可分割的（对与特定任务或大或小）slot。
- **多租户（Multitenancy）**：在某种程度上，YARN 带来的最大的好处是它不仅仅对 MapReduce，也对其它分布式应用敞开了 Hadoop。MapReduce 只是许多 YARN 应用中的一个。甚至可以允许用户在同一个 YARN 集群上运行不同版本的 MapReduce，这让升级 MapReduce 的过程变得可控（Note. MapReduce 的某些部分，例如 job history server 和 shuffle handler，包括 YARN 本身，都需要在整个集群中升级）。

#### 2.2、新旧 MapReduce API 比较

新的 MapReduce API 位于 `org.apache.hadoop.mapreduce` 包及其子包中，旧版本的 MapReduce API 位于 `org.apache.hadoop.mapred` 包中。

新的 MapReduce API 倾向于使用抽象类，而不是接口。这样更容易扩展。比如，新的 MapReduce API 中，`Mapper` 和 `Reducer` 都是抽象类。

新版本的 MapReduce API 广泛使用 context （`Mapper.Context`、`Reducer.Context`），允许用户代码与 MapReduce 系统进行通信。比如，`Mapper.Context` 基本上充当了旧版 MapReduce API 的 `OutputCollector` 和 `Reporter` 的角色。

新的 MapReduce API 统一了配置。旧版本的 MapReduce API 使用 `JobConf` 对象进行 job 的配置，它是对 Hadoop `Configuration` 对象的扩展。新版本的 MapReduce API 中所有的作业配置通过 `Configuration` 来完成，job 执行的控制由 `Job` 类负责，而不再使用 `JobClient`。