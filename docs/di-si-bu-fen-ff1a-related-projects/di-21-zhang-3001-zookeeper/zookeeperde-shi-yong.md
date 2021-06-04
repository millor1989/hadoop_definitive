#### 1、配置服务

分布式应用所需要的最基本的服务是配置服务，通过配置服务集群中的机器可以共享通用的配置信息。在最简单的级别，ZooKeeper可以作为配置的一个高可用的存储，应用可以从ZK获取和更新配置文件。使用ZK的watches可以创建一个活跃的配置服务，通知相关客户端配置的改变。

##### 例

需要保存的配置值是字符串，keys是znode路径，使用znode保存每个键值对；只有一个客户端执行配置的更新。这个配置服务模型与master类似（master更新它的工作者的信息）。

###### *ActiveKeyValueStore*，读写配置

```java
public class ActiveKeyValueStore extends ConnectionWatcher {
	private static final Charset CHARSET = Charset.forName("UTF-8");
	public void write(String path, String value) throws InterruptedException,
		KeeperException {
        // 判断znode是否存在，如果不存在就创建新的znode，存在就进行更新
		Stat stat = zk.exists(path, false);
		if (stat == null) {
			zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
				CreateMode.PERSISTENT);
		} else {
			zk.setDate(path, value.getBytes(CHARSET), -1);
		}
	}

    public String read(String path, Watcher watcher) throws InterruptedException,
    	KeeperException {
            // stat表示一个Stat对象，getData方法会给stat实例传递znode的metadata
            // 此处对znode的metadata不感兴趣，所以传入null
            byte[] data = zk.getData(path, watcher, null/*stat*/);
            return new String (data, CHARSET);
        }
}
```

需要注意的是，把字符串值转换为字节数组。

###### *ConfigUpdater*一个修改配置的应用

```java
public class ConfigUpdater {
    public static final String PAHT = "/config";
    
    public ActiveKeyValueStore store;
    private Random random = new Random();
    
    public ConfigUpdater(String hosts) throws IOException, InterruptedException {
        store = new ActiveKeyValueStore();
        // 连接到ZooKeeper
        store.connect(hosts);
    }

    public void run() throws InterruptedException, KeeperException {
        while (true) {
            String value = random.nextInt(100) + "";
            store.write(PATH, value);
            System.out.printf("Set %s to %s\n", PATH, value);
            TimeUnit.SECONDS.sleep(random.nextInt(10));
        }
    }

    public static void main(String[] args) throws Exception {
        ConfigUpdate configUpdater = new ConfigUpdater(args[0]);
        configUpdater.run();
    }
}
```

###### *ConfigWatcher*读取配置

```java
public class ConfigWatcher implements Watcher {
    private ActiveKeyValueStore store;
    
    public ConfigWatcher(String hosts) throws IOException, InterruptedException {
        store = new ActiveKeyValueStore();
        store.connect(hosts);
    }

    public void displayConfig() throws InterrputedException, KeeperException {
        // 把自己（this）作为Watcher传递
        String value = store.read(ConfigUpdater.PATH, this);
        System.out.printf("Read %s as %s \n", ConfigUpdater.PATH, value);
    }

    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == EventType.NodeDataChanged) {
            try {
                displayConfig();
            } catch (InterruptedException e) {
                System.err.println("Interrupted. Exiting.");
                Thread.currentThread().interrupt();
            } catch (KeeperException e) {
                System.err.printf("KeeperException: %s. Exiting.\n", e);
            }
        }
    }

    public static void main(String[] args) throws Exception {
        ConfigWatcher configWatcher = new ConfigWatcher(args[0]);
        configWatcher.displayConfig();
        // 保持活跃，直到进程被杀死或者线程中断
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

当`ConfigUpdater`更新znode，ZooKeeper会让watcher触发一个`EventType.NodeDataChanged`类型的事件，`ConfgWatcher`的`process()`方法会对这个事件做出反应——读取并展示最新的配置。

因为watch是一次性的信号，因此每次调用`ActiveKeyValueStore`的`read()`方法都传递一个新的watch，这能确保看到未来的更新。但是，这不能保证收到每次的更新，因为znode可能会在watch事件接收和下次读取之间被更新，而这期间客户端没有注册watch进而得不到通知。对于配置服务来说，这不是问题，客户端只关心某个属性的最新值，最新值比之前的值优先级更高。但是，还是要知道这点潜在的限制。

#### 2、弹性ZooKeeper应用

在生产网络环境，依赖可靠的网络环境的应用可能会失败多次。通过检查可能的失败模式，可以采取措施使应用在面对故障时具有弹性。

ZooKeeper Java API每个操作都在`throws`子句中声明了两种类型的异常：`InterruptedException`和`KeeperException`。

##### `InterruptedException`

当操作被打断时会抛出`InterruptedException`。有一种取消阻塞操作的标准Java机制，即调用运行阻塞方法的线程的`interrupt()`方法。成功取消的结果是抛出`InterruptedException`。ZooKeeper也采用了这种标准，可以用这种方式取消ZooKeeper操作。使用ZooKeeper的类或者库通常应该传播`InterruptedException`，以便客户端可以取消操作。`InterruptedException`并不表示存在故障，但是期望操作能够取消，所以在配置服务的例子中将这个异常传播以结束应用。

##### `KeeperException`

如果ZooKeeper服务器发出错误信号或者与服务器的通信有问题会抛出`KeeperException`。对于不同的错误，会有不同的`KeeperException`子类。比如`KeeperException.NoNodeException`是`KeeperException`的一个子类——用于表示执行操作的znode不存在。

每个`KeeperException`的子类都有一个对应的带有类型错误信息的code。比如，`KeeperException.NoNodeException`对应的code是`KeeperException.Code.NONODE`（一个枚举值）。

处理`KeeperException`有两种方式：捕捉`KeeperException`并根据异常code来决定采取什么补救措施；捕捉等价的`KeeperException`子类在每个catch代码块中执行对应的行为。

`KeeperException`有三大类：

- 状态异常（State exceptions）

  当操作因为无法应用到znode树而失败时会发生状态异常。状态异常的产生通常是因为另一个进程同时在修改znode。比如，如果znode首先被另一个进程改变，带版本号的`setData`操作会因为版本号不匹配而失败于`KeeperException.BadVersionException`。用户应该时刻谨记这种类型冲突的可能性并且编码来进行处理。

  某些状态异常预示着程序的错误，比如`KeeperException.NoChildrenForEphemeralsException`是试图为暂时znode创建子znode时抛出的异常。

- 可恢复异常（Recoverable exceptions）

  可恢复异常是指那些应用可以在相同ZK会话中恢复的异常。`KeeperException.ConnectionLossException`意味着ZK连接丢失。ZK会尝试重连，大多数情况下重连可以成功并能确保会话完整。

  但是，ZK不能分辨因为`KeeperException.ConnectionLossException`而失败的操作是否已经应用了。这是部分失败的例子，应该由用户来处理这些不确定性，应该根据应用来决定采取什么措施。

  在这一点上，区分操作是幂等（***idempotent***）还是非幂等（***nonidempotent***）就有用了。幂等的操作是应用一次或者多次结果都是相同的操作，比如读请求和无条件的`setData`。这些操作可以简单地重试。

  非幂等操作不能不假思索地重试，因为应用一次的结果和应用多次的结果是不同的。用户可以将信息编码到znode的路径名或者它的数据来检测更新是否被应用了。锁服务实现中继续讨论非幂等操作的失败恢复问题。

- 不可恢复异常（Unrecoverable exceptions）

  某些情况下，ZK会话会失效——会话超时或者会话关闭（这两种情况下会抛出`KeeperException.SessionExpiredException`）、或者认证失败（`KeeperException.AuthFailedException`）。无论哪种情况，与会话相关的所有暂时节点都会丢失，重新连接到ZK之前应用需要重建它的状态。

##### 2.1、可靠的配置服务

之前配置服务例子中`ActiveKeyValueStore`的`write()`方法是幂等的，所以可以无条件的重试。

```java
public void write(String path, String value) throws InterruptedException,
	KeeperException {
	int retries = 0;
	while (true) {
		try {
            Stat stat = zk.exists(path, false);
            if (stat == null) {
                zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
                         CreateMode.PERSISTENT);
            } else {
                zk.setData(path, value.getBytes(CHARSET), stat.getVersion());
            }
            return ;
        } catch (KeeperException.SessionExpiredException e) {
            // 会话过期，ZooKeeper对象进入CLOSED状态，不能重连，所以不重试。将异常抛出。
            // 调用者可以创建一个新的ZooKeeper实例，然后重试
            throw e;
        } catch (KeeperException e) {
            if (retries++ == MAX_RETRIES) {
                throw e;
            }
            // 休眠然后重试
            TimeUnit.SECONDS.sleep(RETRY_PERIOD_SECONDS);
        }
	}
}
```

会话过期时，创建新的会话，用`ResilientConfigUpdater`（重命名后的`ConfigUpdater`）：

```java
public static void main(String[] args) throws Exception {
    while (true) {
        try {
            ResilientConfigUpdater configUpdater =
                new ResilientConfigUpdater(args[0]);
            configUpdater.run();
        } catch (KeeperException.SessionExpiredException e) {
            // 空语句块，因为在循环中，会话过期，自动进入新一次的循环——创建新的会话
        } catch (KeeperException e) {
            // 已经重试，如果重试后依然失败直接退出
            e.printStackTrace();
            break;
        }
    }
}
```

还可以通过查看watcher（`ConnectionWatcher`）中`KeeperState`的类型是否是`Expired`来判断会话是否过期。此时，即使发生`KeeperException.SessionExpiredException`，只用重试`write()`方法就行了，因为最终都会重建连接。不管采用哪种机制从过期会话恢复，重要的是不同的连接丢失类型要进行不同的处理。

这只是一种重试处理。还有许多其它的重试处理，比如使用指数退让（exponential backoff）——每次重试直接的时间间隔都乘以一个常数。

其实还有另外一种故障模式，创建`ZooKeeper`对象时，它会尝试连接到ZK服务器，如果连接失败或者超时，它会尝试团体中的其它服务器。如果尝试了所有的服务器仍然不能连接，则抛出`IOException`。所有的ZK服务器都不可用的可能性很低，但是，某些应用可能会选择循环地重试直到ZK可用。

#### 3、锁服务（Lock Service）

**分布式锁**是一种保证一组进程间互斥（mutual exclusion）的机制。同一时间，只有一个进程可以持有锁。分布式锁可以用于大型分布式系统的头领选择，头领进程总是持有锁。

**注意**：不要将ZooKeeper的头领选择和一般的头领选择混淆。ZK的头领选择是不公开的，用于需要与master进程保持一致的分布式系统。

为了使用ZK实现分布式锁，需要使用顺序znodes来为争夺锁的进程分配一个顺序。想法很简单：首先，指定一个锁znode，一般描述要上锁的entity（比如，*/leader*）；然后，想要获得锁的客户端创建作为锁znode子节点的顺序的暂时的znodes。同一时间，具有最小序号的客户端持有锁。例如，两个客户端同时在*/leader/lock-1*和*/leader/lock-2*创建了znodes，那么创建*/leader/lock-1*的客户端持有锁，因为它的序号最小。ZK服务是顺序的决定者，因为它负责分配序号。通过删除znode */leader/lock-1*可以释放锁；或者，如果客户端进程退出，因为*/leader/lock-1*是暂时的znode它也会被删除。随后，创建*/leader/lock-2* znode的客户端将持有锁，因为它的序号是余下的znodes中最小的。通过创建一个在持有锁的znodes消失时触发的watch可以确保能够得到通知。

分布式锁的**实现**伪代码（pseudocode）：

1. 在锁znode下创建锁以*/lock-*形式命名的暂时的顺序znode，并且将它的实际路径名（`create`操作的返回值）保存。
2. 获取锁znode的子znodes并设置一个watch。
3. 如果第`1`步创建的znode的路径名是第`2`步获得子znode中序号最小的，那么它将获取锁。退出。
4. 等待第`2`步设置的watch的通知，再次执行第`2`步。

##### 3.1、羊群效应（The herd effect）

虽然以上分布式锁的实现算法是正确的，但是存在一些问题。首先是，*羊群效应*。如果有成百上千的客户端，都尝试获取锁，每个客户端都在锁znode放置一个等待它的子znode变化的watch。每当锁释放或者其它的进程启动了锁获取进程，都会触发watch，每个客户端都会收到通知。而实际上只有一个客户端可以成功获取锁，但是维护和发送watch事件到客户端的进程会引发交通高峰（traffic spikes），会对ZK服务器造成压力。

要避免羊群效应，需要优化通知条件。实现分布式锁的关键观测是，当拥有当前最小序号的子znode消失而不是任意子znode删除（或者创建）时通知客户端。在之前的例子中，如果客户端已经创建了znode */leader/lock-1*，*/leader/lock-2*，*/leader/lock-3*，持有*/leader/lock-3*的客户端只应该在*/leader/lock-2*消失时收到通知。当*/leader/lock-1*消失或者*/leader/lock-4*创建时，它都不必收到通知。

##### 3.2、可恢复异常（Recoverable exceptions）

以上分布式锁算法的另一个问题是：没有处理因为连接丢失而导致的创建操作失败。要知道，这种情况下不知道创建操作成功还是失败。创建一个顺序znode是一个非幂等的操作，不能简单地重试，因为如果之前的`create`成功了将会存在一个无法删除的（至少要等到客户端会话结束）孤立znode。死锁是一个糟糕的情形。

问题是重连后，客户端不能区分它是否创建了子znodes。可以在znode名称中嵌入一个唯一身份，如果连接丢失，可以检查锁znode的所有子znode中是否有包含特定唯一身份的子znode。如果有，那么`create`操作已经成功了，如果没有那么就可以放心的创建一个新的顺序子znode。

客户端的唯一身份是一个long整数，对于ZK服务来说是唯一的，因而用来区分客户端处理连接丢失非常理想。调用`ZooKeeper` Java类的`getSessionId()`方法即可获取会话唯一身份。

暂时的顺序znode应该使用*lock-&lt;sessionId>-*形式的名称，这样ZK可以将序号追加到后面，znode最终的名字则会是*lock-&lt;sessionId>-&lt;sequenceNumber>*。序号对父节点来说是唯一的，使用这种方法，子znode既可以区分它们的创建者，也具有了创建的顺序。

##### 3.3、不可恢复异常（Unrecoverable exceptions）

如果客户端的ZK会话过期，客户端创建的暂时的znode会被删除，持有的锁会很快释放（或者至少会丧失客户端获取锁的顺序）。使用锁的应用应该能够意识到它不再持有锁了，清除它的状态，然后创建一个新的锁对象并且尝试获取锁。注意，是应用（而不是锁的实现）控制这个过程，锁的实现是不知道应用该如何清除状态的。

##### 3.4、实现

要考虑到所有的失败类型是很复杂，所以正确地实现分布式锁是一个精细的工程。ZK Java自带了一个生成级的（production-quality）锁实现`WriteLock`，客户端很简单就能使用。

#### 4、其它的分布式数据结构和协议

可以用ZK构建很多其它的分布式数据结构和框架，比如，barriers、queues、和two-phase commit；但是这些都是同步协议，尽管使用的是异步的ZK操作（比如，通知）来构建它们。

[ZK官网](https://zookeeper.apache.org/)用伪代码的形式描述了这些数据结构和协议。ZK自带了其中某些标准用途的实现（包括locks，leader election，和queues）；可以在ZK distribution的`recipes`目录中找到这些实现。

[Apache Curator项目](http://curator.apache.org/)也提供了一组ZK用途的扩展，还包括一个简化的ZK客户端。

##### 4.1、BookKeeper和Hedwig

BookKeeper是一个高可用的并且可靠的日志服务，可以用来提供***WAL***（write-ahead logging，确保存储系统中数据完整性的常用技术）。在使用WAL的系统中，每次写操作被实施之前都被写到事务日志（transaction log）中。使用这种方式，不必再每次写操作之后将数据写到持久化存储，因为系统故障时，通过重播事务日志可以恢复没有实施的写操作的最新状态。

BookKeeper创建日志的客户端叫作*ledgers*，追加到*ledger*的每条记录叫作*ledger entry*——它只是一个字节数组。*ledgers*由*bookies*（备份*ledger*数据的服务器）管理。*ledger*的数据不保存在ZooKeeper中，只有metadata保存在ZooKeeper中。

一般，挑战是使使用WAL的系统在写事务日志的节点故障时保持健壮。通常通过以某种方式备份事务日志达成。HDFS是高可用的，使用一组journal节点来提供高可用的edit log。尽管与BookKeeper类似，但是有专门写HDFS的服务，并且不使用ZK作为协调引擎。

*Hedwig*是一个基于BookKeeper构建的、基于topic的发布订阅系统。因为底层使用ZK，Hedwig是一个高可用的服务，并且能够保证消息的传送（即使订阅者离线很长时间）。

可以在[bookkeeper官网](http://bookkeeper.apache.org/)了解更多。