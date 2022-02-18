### HA架构

<img src="/Users/licheng/Documents/Typora/Picture/HDFS HA架构实现.png" style="zoom: 67%;" />

* Active NameNode 和 Standby NameNode：两台 NameNode 形成互备，一台处于 Active 状态，为主 NameNode，另外一台处于 Standby 状态，为备 NameNode，只有主 NameNode 才能对外提供读写服务
* ZKFailoverController (主备切换控制器，FC)：ZKFailoverController 作为独立的进程运行，对 NameNode 的主备切换进行总体控制。ZKFailoverController 能及时检测到 NameNode 的健康状况，在主 NameNode 故障时借助 Zookeeper 实现自动的主备选举和切换 (当然 NameNode 目前也支持不依赖于 Zookeeper 的手动主备切换)。
* Zookeeper 集群：为主备切换控制器提供主备选举支持。
* 共享存储系统：共享存储系统是实现 NameNode 的高可用最为关键的部分，共享存储系统保存了 NameNode 在运行过程中所产生的 HDFS 的元数据。主 NameNode 和备 NameNode 通过共享存储系统实现元数据同步。在进行主备切换的时候，新的主 NameNode 在确认元数据完全同步之后才能继续对外提供服务。
* DataNode 节点：因为主 NameNode 和备 NameNode 需要共享 HDFS 的数据块和 DataNode 之间的映射关系，为了使故障切换能够快速进行，DataNode 会同时向主 NameNode 和备 NameNode 上报数据块的位置信息。

### NameNode 的主备切换实现

NameNode 主备切换主要由 ZKFailoverController、HealthMonitor 和 ActiveStandbyElector 这 3 个组件来协同实现。

ZKFailoverController 作为 NameNode 机器上一个独立的进程启动 (在 HDFS 启动脚本之中的进程名为 zkfc)，启动的时候会创建 HealthMonitor 和 ActiveStandbyElector 这两个主要的内部组件。HealthMonitor 主要负责检测 NameNode 的健康状态。ActiveStandbyElector 主要负责完成自动的主备选举。

<img src="/Users/licheng/Documents/Typora/Picture/NameNode主备选举流程.png" style="zoom: 50%;" />

Namenode (包括 YARN ResourceManager) 的主备选举是通过 ActiveStandbyElector 来完成的，ActiveStandbyElector 主要是利用了 Zookeeper 的写一致性和临时节点机制，具体的主备选举实现如下：

#### 创建锁节点

1. 如果刚开始运行，还没有进行过主备选举的话，那么相应的 ActiveStandbyElector 就会发起一次主备选举，尝试在 Zookeeper 上创建一个路径为 /hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 的临时节点。(${dfs.nameservices} 为 Hadoop 的配置参数 dfs.nameservices 的值，下同)。
2. Zookeeper 的写一致性会保证最终只会有一个 ActiveStandbyElector 创建成功，那么创建成功的 ActiveStandbyElector 对应的 NameNode 就会成为主 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Active 状态。
3. 创建失败的 ActiveStandbyElector 对应的 NameNode 成为备 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Standby 状态。

#### 注册 Watcher 监听

不管创建 /hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock 节点是否成功，ActiveStandbyElector 随后都会向 Zookeeper 注册一个 Watcher 来监听这个节点的状态变化事件，ActiveStandbyElector 主要关注这个节点的 NodeDeleted 事件。

#### 自动触发主备选举

如果 Active NameNode 对应的 HealthMonitor 检测到 NameNode 的状态异常时， ZKFailoverController 会主动删除当前在 Zookeeper上建立的临时节点/hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock，这样处于 Standby 状态的 NameNode 的 ActiveStandbyElector 注册的监听器就会收到这个节点的 NodeDeleted 事件。

> HealthMonitor 主要检测 NameNode 的两类状态，分别是 HealthMonitor.State 和 HAServiceStatus。HealthMonitor.State 反映了 NameNode 节点的健康状况，主要是磁盘存储资源是否充足。HAServiceStatus 主要反映的是 NameNode 的 HA 状态。

ActiveStandbyElector 收到事件后，会再次进入到建/hadoop-ha/​{dfs.nameservices}/ActiveStandbyElectorLock 节点的流程，如果创建成功，这个本来处于 Standby 状态的 NameNode 就选举为主 NameNode 并随后开始切换为 Active 状态。

当然，如果是 Active 状态的 NameNode 所在的机器整个宕掉的话，那么根据 Zookeeper 的临时节点特性，/hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock 节点会自动被删除，从而也会自动进行一次主备切换。

### HDFS 脑裂问题

在实际中，NameNode 可能会出现这种情况，NameNode 在垃圾回收时，可能会在长时间内整个系统无响应，因此，也就无法向 zk 写入心跳信息，这样的话可能会导致临时节点掉线，备 NameNode 会切换到 Active 状态，这种情况，可能会导致整个集群会有同时有两个 NameNode，这就是脑裂问题。
脑裂问题的解决方案是隔离 (Fencing)，主要是在以下三处采用隔离措施：

* 共享存储：任一时刻，只有一个 NameNode 可以写入；
* DataNode：需要保证只有一个 NameNode 发出与管理数据副本有关的删除命令；
* Client：需要保证同一时刻只有一个 NameNode 能够对 Client 的请求发出正确的响应。

#### 具体实现

ActiveStandbyElector 为了实现 fencing，在成功创建 Zookeeper 节点 hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock，成为 Active NameNode 之后，创建另外一个路径为 /hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb 的持久节点，这个节点里面保存了这个 Active NameNode 的地址信息。Active NameNode 的 ActiveStandbyElector 在正常的状态下关闭 Zookeeper Session 的时候，会一起删除这个持久节点。

如果 ActiveStandbyElector 在异常的状态下关闭 Zookeeper Session (比如前述的 Zookeeper 假死)，那么由于 /hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb 是持久节点，会一直保留下来，后面当另一个 NameNode 选主成功之后，会注意到上一个 Active NameNode 遗留下来的这个节点，从而会回调 ZKFailoverController 的方法对旧的 Active NameNode 进行 fencing。

在进行 fencing 的时候，首先尝试调用这个旧 Active NameNode 的 HAServiceProtocol RPC 接口的 transitionToStandby 方法，看能不能把它转换为 Standby 状态；如果 transitionToStandby 方法调用失败，那么就执行 Hadoop 配置文件之中预定义的隔离措施。只有在成功地执行完成 fencing 之后，选主成功的 ActiveStandbyElector 才会回调 ZKFailoverController 的 becomeActive 方法将对应的 NameNode 转换为 Active 状态，开始对外提供服务。

>Hadoop 目前主要提供两种隔离措施，通常会选择第一种：
>
>* sshfence：通过 SSH 登录到目标机器上，执行命令 fuser 将对应的进程杀死；
>* shellfence：执行一个用户自定义的 shell 脚本来将对应的进程隔离。

### 共享存储

<img src="/Users/licheng/Documents/Typora/Picture/基于QJM的共享存储系统架构图.png" style="zoom: 67%;" />

#### 基于 QJM 的共享存储的数据同步机制

基于 QJM 的共享存储系统主要用于保存 EditLog，并不保存 FSImage 文件。FSImage 文件还是在 NameNode 的本地磁盘上。

QJM 共享存储的基本思想来自于 Paxos 算法，采用多个称为 JournalNode 的节点组成的 JournalNode 集群来存储 EditLog。每个 JournalNode 保存同样的 EditLog 副本。每次 NameNode 写 EditLog 的时候，除了向本地磁盘写入 EditLog 之外，也会并行地向 JournalNode 集群之中的每一个 JournalNode 发送写请求，只要大多数 (majority) 的 JournalNode 节点返回成功，就认为向 JournalNode 集群写入 EditLog 成功。

#### 基于 QJM 的共享存储系统的内部实现架构图

<img src="/Users/licheng/Documents/Typora/Picture/基于QJM的共享存储系统内部实现架构图.png" style="zoom:67%;" />

##### FSEditLog

这个类封装了对 EditLog 的所有操作，是 NameNode 对 EditLog 的所有操作的入口。

##### JournalSet

这个类封装了对本地磁盘和 JournalNode 集群上的 EditLog 的操作，内部包含了两类 JournalManager：

* FileJournalManager：用于实现对本地磁盘上 EditLog 的操作。
* QuorumJournalManager：用于实现对 JournalNode 集群上共享目录的 EditLog 的操作。

FSEditLog 只会调用 JournalSet 的相关方法，不会直接使用 FileJournalManager 和QuorumJournalManager。

##### FileJournalManager

封装了对本地磁盘上的 EditLog 文件的操作，不仅 NameNode 在向本地磁盘上写入 EditLog 的时候使用 FileJournalManager，JournalNode 在向本地磁盘写入 EditLog 的时候也复用了 FileJournalManager 的代码和逻辑。

##### QuorumJournalManager

封装了对 JournalNode 集群上的 EditLog 的操作，它会根据 JournalNode 集群的 URI 创建负责与 JournalNode 集群通信的类 AsyncLoggerSet， QuorumJournalManager 通过 AsyncLoggerSet 来实现对 JournalNode 集群上的 EditLog 的写操作，对于读操作，QuorumJournalManager 则是通过 Http 接口从 JournalNode 上的 JournalNodeHttpServer 读取 EditLog 的数据。

##### AsyncLoggerSet

内部包含了与 JournalNode 集群进行通信的 AsyncLogger 列表，每一个 AsyncLogger 对应于一个 JournalNode 节点，另外 AsyncLoggerSet 也包含了用于等待大多数 JournalNode 返回结果的工具类方法给 QuorumJournalManager 使用。

##### AsyncLogger

具体的实现类是 IPCLoggerChannel，IPCLoggerChannel 在执行方法调用的时候，会把调用提交到一个单线程的线程池之中，由线程池线程来负责向对应的 JournalNode 的 JournalNodeRpcServer 发送 RPC 请求。

##### JournalNodeRpcServer

运行在 JournalNode 节点进程中的 RPC 服务，接收 NameNode 端的 AsyncLogger 的 RPC 请求。

##### JournalNodeHttpServer

运行在 JournalNode 节点进程中的 Http 服务，用于接收处于 Standby 状态的 NameNode 和其它 JournalNode 的同步 EditLog 文件流的请求。

#### Active NameNode 提交 EditLog 到 JournalNode 集群

* 当处于 Active 状态的 NameNode 调用 FSEditLog 类的 logSync 方法来提交 EditLog 的时候，会通过 JouranlSet 同时向本地磁盘目录和 JournalNode 集群上的共享存储目录写入 EditLog。
* 写入 JournalNode 集群是通过并行调用每一个 JournalNode 的 QJournalProtocol RPC 接口的 journal 方法实现的，如果对大多数 JournalNode 的 journal 方法调用成功，那么就认为提交 EditLog 成功，否则 NameNode 就会认为这次提交 EditLog 失败。
* 提交 EditLog 失败会导致 Active NameNode 关闭 JournalSet 之后退出进程，留待处于 Standby 状态的 NameNode 接管之后进行数据恢复。

#### Standby NameNode 从 JournalNode 集群同步 EditLog

当 NameNode 进入 Standby 状态之后，会启动一个 EditLogTailer 线程和 StandbyCheckpointer 线程。EditLogTailer 线程会定期调用 EditLogTailer 类的 doTailEdits 方法从 JournalNode 集群上同步 EditLog，然后把同步的 EditLog 回放到内存之中的文件系统镜像上 (并不会同时把 EditLog 写入到本地磁盘上)。

> StandbyCheckpointer 线程的作用其实是为了替代 Hadoop 1.x 版本之中的 Secondary NameNode 的功能，StandbyCheckpointer 线程会在 Standby NameNode 节点上定期进行 Checkpoint，将 Checkpoint 之后的 FSImage 文件上传到 Active NameNode 节点。

### HDFS 2.0 Federation 实现

在 1.0 中，HDFS 的架构设计有以下缺点：

* namespace 扩展性差：在单一的 NN 情况下，因为所有 namespace 数据都需要加载到内存，所以物理机内存的大小限制了整个 HDFS 能够容纳文件的最大个数(namespace 指的是 HDFS 中树形目录和文件结构以及文件对应的 block 信息)。

* 性能可扩展性差：由于所有请求都需要经过 NN，单一 NN 导致所有请求都由一台机器进行处理，很容易达到单台机器的吞吐。

* 隔离性差：多租户的情况下，单一 NN 的架构无法在租户间进行隔离，会造成不可避免的相互影响。

  <img src="/Users/licheng/Documents/Typora/Picture/image-20200501211336583.png" alt="image-20200501211336583" style="zoom:67%;" />

**Federation 的核心思想**是将一个大的 namespace 划分多个子 namespace，并且每个 namespace 分别由单独的 NameNode 负责，这些 NameNode 之间互相独立，不会影响，不需要做任何协调工作(其实跟拆集群有一些相似)，集群的所有 DataNode 会被多个 NameNode 共享。

其中，每个子 namespace 和 DataNode 之间会由数据块管理层作为中介建立映射关系，数据块管理层由若干数据块池(Pool)构成，每个数据块只会唯一属于某个固定的数据块池，而一个子 names4pace 可以对应多个数据块池。每个 DataNode 需要向集群中所有的 NameNode 注册，且周期性地向所有 NameNode 发送心跳和块报告，并执行来自所有 NameNode 的命令。

* 一个 block pool 由属于同一个 namespace 的数据块组成，每个 DataNode 可能会存储集群中所有 block pool 的数据块。
* 每个 block pool 内部自治，也就是说各自管理各自的 block，不会与其他 block pool 交流，如果一个 NameNode 挂掉了，不会影响其他 NameNode。
* 某个 NameNode 上的 namespace 和它对应的 block pool 一起被称为 namespace volume，它是管理的基本单位。当一个 NameNode/namespace 被删除后，其所有 DataNode 上对应的 block pool 也会被删除，当集群升级时，每个 namespace volume 可以作为一个基本单元进行升级。

https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/