### YARN架构

<img src="/Users/licheng/Documents/Typora/Picture/Yarn架构图.png" style="zoom:67%;" />

#### ResourceManager

ResourceManager 通常在独立的机器上以后台进程的形式运行，它是整个集群资源的主要协调者和管理者。ResourceManager 负责给用户提交的所有应用程序分配资源，它根据应用程序优先级、队列容量、ACLs、数据位置等信息，做出决策，然后以共享的、安全的、多租户的方式制定分配策略，调度集群资源。
#### NodeManager

NodeManager 是 YARN 集群中的每个具体节点的管理者。主要负责该节点内所有容器的生命周期的管理，监视资源和跟踪节点健康。具体如下：
* 启动时向 ResourceManager 注册并定时发送心跳消息，等待 ResourceManager 的指令；
* 维护 Container 的生命周期，监控 Container 的资源使用情况；
* 管理任务运行时的相关依赖，根据 ApplicationMaster 的需要，在启动 Container 之前将需要的程序及其依赖拷贝到本地。

#### ApplicationMaster

在用户提交一个应用程序时，YARN 会启动一个轻量级的进程ApplicationMaster。ApplicationMaster 负责协调来自 ResourceManager 的资源，并通过 NodeManager 监视容器内资源的使用情况，同时还负责任务的监控与容错。具体如下：
* 根据应用的运行状态来决定动态计算资源需求；
* 向 ResourceManager 申请资源，监控申请的资源的使用情况；
* 跟踪任务状态和进度，报告资源的使用情况和应用的进度信息；
* 负责任务的容错。

#### Container

Container 是 YARN 中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。当 AM 向 RM 申请资源时，RM 为 AM 返回的资源是用 Container 表示的。YARN 会为每个任务分配一个 Container，该任务只能使用该 Container 中描述的资源。ApplicationMaster 可在 Container 内运行任何类型的任务。例如，MapReduce ApplicationMaster 请求一个容器来启动 map 或 reduce 任务，而 Giraph ApplicationMaster 请求一个容器来运行 Giraph 任务。

### 调度器

#### FIFO

默认的调度器，将所有的任务放到队列中，按照任务优先级和到达时间的先后顺序为每个任务分配资源。如果第一个任务需要的资源被满足了，查看剩下的资源是否还满足第二个任务，如果也满足，就为它分配资源，然后继续往下判断。

优点：简单，容易配置。

缺点：如果某个任务特别大的话，后面的作业等待时间会很长。

#### CapacityScheduler

用于一个集群中运行多个 Application 的情况，目标是最大化吞吐量和集群利用率。

CapacityScheduler 允许将整个集群的资源分成多个部分，每个组织使用其中的一部分，即每个组织有一个专门的队列，每个组织的队列还可以进一步划分成层次结构，从而允许组织内部的不同用户组的使用。

比如一个队列用于小作业执行，只占集群资源的 20%，一个队列用于执行大作业，占用集群资源的 80%。

#### FairScheduler

任务公平的共享所有的资源。

<img src="/Users/licheng/Documents/Typora/Picture/image-20200827165043789.png" alt="image-20200827165043789" style="zoom:50%;" />

两个用户 A 和 B。A 提交 job1 时集群内没有正在运行的 app，因此 job1 独占集群中的资源。用户 B 的 job2 提交时，job2 在 job1 释放一半的 containers 之后，开始执行。job2 还没执行完的时候，用户 B 提交了 job3，job2 释放它占用的一半 containers 之后，job3 获得资源开始执行。