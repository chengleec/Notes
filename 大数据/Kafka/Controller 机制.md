### Controller 职责

* 监听 Broker 相关的变化。
  
  * 监听 /Brokers/ids/ 节点，用于 Broker 上下线的监听。
* 监听 Partition 相关的变化。
  * 监听 /admin/reassign_partitions 节点，用于分区副本迁移的监听。
  * 监听 /isr_change_notification 节点，用于分区 ISR 集合变更的监听。
  * 监听 /admin/preferred-replica-election 节点，用与 Partition 最优 leader 选举的监听。
* 监听 topic 相关的变化。
  * 监听 /Brokers/topics 节点，用于 topic 的新建的监听。
  * 监听 /Brokers/topics/[topic] 节点，用于 topic 分区变化的监听。
  
  * 监听 /admin/delete_topics 节点，用于 topic 删除的监听。
* 启动并管理分区状态机和副本状态机。
* 更新集群的元数据信息。

### Controller 选举过程

每个 Broker 启动的时候会去尝试去读取 /controller 节点的 Brokerid 的值，如果读取到 Brokerid 的值不为 -1，则表示已经有其它 Broker 节点成功竞选为 Controller，所以当前 Broker 就会放弃竞选；

如果 Zookeeper 中不存在 /controller 这个节点，或者这个节点中的数据异常，那么就会尝试去创建 /controller 这个节点，创建成功的那个 Broker 就会成为 Controller，而创建失败的 Broker 则表示竞选失败。每个 Broker 都会在内存中保存当前 Controller 的 Brokerid 值，这个值可以标识为 activeControllerId。

每个和 Controller 交互的请求都会携带上 controller_epoch 这个字段，如果请求的 controller_epoch 值小于内存中的 controller_epoch 值，则认为这个请求是向已经过期的 Controller 所发送的请求，那么这个请求会被认定为无效的请求。如果请求的 controller_epoch 值大于内存中的 controller_epoch 值，那么则说明已经有新的 Controller 当选了。

由此可见，Kafka 通过 controller_epoch 来保证 Controller 的唯一性，进而保证相关操作的一致性。

### 分区状态机

<img src="/Users/licheng/Documents/Typora/Picture/image-20200629183304418.png" alt="image-20200629183304418" style="zoom:67%;" />

* NonExistentPartition：这个代表着这个 Partition 之前没有被创建过或者之前创建了现在又被删除了，它有效的前置状态是 OfflinePartition；

* NewPartition：Partition 创建后，它将处于这个状态，这个状态的 Partition 还没有 leader 和 isr，它有效的前置状态是 NonExistentPartition；

* OnlinePartition：一旦这个 Partition 的 leader 被选举出来了，它将处于这个状态，它有效的前置状态是 NewPartition、OnlinePartition、OfflinePartition；

* OfflinePartition：如果这个 Partition 的 leader 掉线，这个 Partition 将被转移到这个状态，它有效的前置状态是 NewPartition、OnlinePartition、OfflinePartition。

### 副本状态机

<img src="/Users/licheng/Documents/Typora/Picture/image-20200629183613543.png" alt="image-20200629183613543" style="zoom:67%;" />

* NewReplica：当创建了 topic 或者重分配分区时 Controller 会创建新的副本，就处在这个状态。这种状态下该 Replica 只能作为 follower，它可以是 Replica 删除后的一个临时状态，它有效的前置状态是 NonExistentReplica；
* OnlineReplica：一旦这个 Replica 被分配到指定的 Partition 上，并且 Replica 创建完成，那么它将会被置为这个状态，在这个状态下，这个 Replica 既可以作为 leader 也可以作为 follower，它有效的前置状态是 NewReplica、OnlineReplica 或 OfflineReplica；
* OfflineReplica：如果一个 Replica 挂掉 (所在的节点宕机或者其他情况)，该 Replica 将会被转换到这个状态，它有的效前置状态是 NewReplica、OfflineReplica 或者 OnlineReplica；
* ReplicaDeletionStarted：Replica 开始删除时被置为的状态，它有效的前置状态是 OfflineReplica；
* ReplicaDeletionSuccessful：如果 Replica 在删除时没有遇到任何错误信息，它将被置为这个状态，这个状态代表该 Replica 的数据已经从节点上清除了，它有效的前置状态是 ReplicaDeletionStarted；
* ReplicaDeletionIneligible：如果 Replica 删除失败，它将会转移到这个状态，这个状态意思是非法删除，也就是删除是无法成功的，它有效的前置状态是 ReplicaDeletionStarted；
* NonExistentReplica：如果 Replica 删除成功，它将被转移到这个状态，它有效的前置状态是：ReplicaDeletionSuccessful。

### Topic 的新建、扩容与删除

![img](https://matt33.com/images/kafka/topic-create-alter.png)

#### 新建 Topic

当我们使用命令行新建一个 topic (topic_test) 时，Kafka 会在 `/brokers/topics` 下新建一个节点信息。当 `/Brokers/topics` 中的子节点发生变化时，会触发 `TopicChangeListener()` 的 `handleChildChange()` 方法。

* 获取 `/Brokers/topics` 路径下的集合，与当前 controller 中保存的 Topic 集合比较，找出新增 Topic 集合。

* 更新 Controller 的 Topic 缓存列表。

* 从 `/Brokers/topics/topic_test` 结点取出这个 topic 所有分区的分配方案，更新 Controller 对应的缓存信息。

* 调用 `KafkaController.onNewTopicCreation()` 方法创建 topic。

  * 注册这个 topic 的 PartitionModificationsListener 监听器；

  * 通过 `onNewPartitionCreation()` 创建该 Topic 的所有 Partition。
    * 将新创建的 Partition 状态置为 NewPartition 状态，此时 Partition 刚刚创建，只是分配了相应的 Replica 但是还没有 leader 和 isr，不能正常工作;
    * 将该 Partition 对应的 Replica 列表状态设置为 NewReplica 状态，这部分只是将 Replica 的状态设置为了 NewReplica，并没有做其他的处理;
    * 将分区状态从 NewPartition 改为 OnlinePartition 状态，初始化 leader 和 isr (让 AR 中第一个 replica 作为 leader，所有可用的 replica 作为 ISR)。并将结果更新到 Zookeeper 和 Controller 的缓存中。最后向每个 replica 发送 LeaderAndIsrRequest，向每个 Broker 发送 UpdateMetadataRequest 请求。
    * 设置副本状态机状态从 NewReplica 装换到 OnlineReplica。

#### Topic 扩容

当使用命令行更新 Topic Partition 信息时，`/Brokers/topics/topic_test`节点会发生变化，会触发`PartitionModificationsListener()` 的 `HandleDataChange()`方法。

* 首先获取该 Topic 在 ZK 的 Partition 副本列表，跟本地的缓存做对比，获取新增的 Partition 列表；

* 检查这个 Topic 是否被标记为删除，如果被标记了，那么直接跳过，不再处理这个 Partition 扩容的请求；

* 调用 `KafkaController.onNewPartitionCreation()` 新建该 Partition (同上)。

#### 删除 Topic

![img](https://matt33.com/images/kafka/topic-delete.png)

通过命令行将 Topic 标记为删除后：

* 首先调用 DeleteTopicsListener 的 HandleChildChange() 方法，然后通过 `enqueueTopicsForDeletion()` 方法将 Topic 添加到要删除的 Topic 列表中；

  * 根据要删除的 Topic 列表，过滤出那些不存在的 Topic 列表，直接从 ZK 中清除(只是从 `/admin/delete_topics` 中移除)；

  * 如果集群不允许 Topic 删除，直接从 ZK 中清除 (只是从 `/admin/delete_topics` 中移除) 这些 Topic 列表，结束流程；

  * 如果这个列表中有正在进行副本迁移或 leader 选举的 Topic，那么先将这些 Topic 加入到 `topicsIneligibleForDeletion` 中，即标记为非法删除；

  * 通过 `enqueueTopicsForDeletion()` 方法将 Topic 添加到要删除的 Topic 列表(`topicsToBeDeleted`)、将 Partition 添加到要删除的 Partition 列表中(`partitionsToBeDeleted`)。

* DeleteTopicsThread 这个线程会不断调用 `doWork()` 方法，这个方法被调用时，它会遍历 `topicsToBeDeleted` 中的所有 Topic 列表，执行删除操作；

  * 遍历所有要删除的 Topic，如果该 Topic 的所有副本都下线成功 (状态为 ReplicaDeletionSuccessful) 时，那么执行 `completeDeleteTopic()` 方法完成 Topic 的删除；

  * 如果 Topic 在删除过程有失败的副本 (状态为 ReplicaDeletionIneligible)，那么执行 `markTopicForDeletionRetry()` 将失败的 Replica 状态设置为 OfflineReplica；

  * 判断 Topic 是否允许删除，允许则调用 `TopicDeletionManager.onTopicDeletion()` 执行 Topic 删除。

* Topic 删除时，会先调用 `onPartitionDeletion()` 方法删除所有的 Partition，然后在 Partition 删除时，执行 `startReplicaDeletion()` 方法删除该 Partition 的副本。

  * 首先获取当前集群所有存活的 Broker 信息，根据这个信息可以知道 Topic 哪些副本所在节点是处于 dead 状态；

  * 找到那些已经成功删除的 Replica 列表 (状态为 ReplicaDeletionSuccessful)，进而可以得到那些还没有成功删除、并且存活的 Replica 列表(`replicasForDeletionRetry`)；

  * 将处于 dead 节点上的 Replica 的状态设置为 ReplicaDeletionIneligible 状态；

  * 然后重新删除 replicasForDeletionRetry 列表中的副本，先将其状态转移为 OfflineReplica，再转移为 ReplicaDeletionStarted 状态(真正从发送 StopReplica +从物理上删除数据)；

  * 如果有 Replica 所在的机器处于 dead 状态，那么将 Topic 设置为非法删除状态。

* 如果 Topic 删除完成(所有 Replica 的状态都变为 ReplicaDeletionSuccessful 状态)，那么就执行 TopicDeletionManager 的 `completeDeleteTopic()` 完成删除流程，即更新状态信息，并将 Topic 的 meta 信息从缓存和 ZK 中清除。

  * 取消对该 Topic 的 partitionModifyListener 监听器；

  * 将状态为 ReplicaDeletionSuccessful 的副本状态都转移成 NonExistentReplica；

  * 将该 Topic Partition 状态先后转移成 OfflinePartition、NonExistentPartition 状态，正式下线了该 Partition；

  * 从分区状态机和副本状态机中移除这个 Topic 记录；

  * 从 Controller 缓存和 ZK 中清除这个 Topic 的相关记录。

### 分区副本迁移

#### 名词解释

{1,2,3} -> {2,3,4}

* AR：assign replica，当前的副本列表，会不断变化。
* RAR：Reassigned replica，新分配过的副本列表，{2,3,4}。
* ORA：Original list of replicas for partition，分区原来的副本列表，{1,2,3}。
* RAR-OAR：需要创建、数据同步的新副本，{4}。
* OAR-RAR：不需要创建、数据同步的副本，{2,3}。

#### 迁移流程

<img src="/Users/licheng/Documents/Typora/Picture/image-20200629213502402.png" alt="image-20200629213502402" style="zoom:67%;" />

当用命令行发起分区重分配操作时，它会在 /admin/reassign_partitions 路径下创建节点，进而触发 PartitionsReassignedListener 的`handleDataChange()` 方法。

* 过滤掉正在执行副本迁移的分区和已经标记为删除的分区。
* 如果 Partition 新分配的 replica 与之前的 replicas 相同，那么不会进行副本迁移，直接抛出异常。
* 将该 Topic 标记为不可删除状态，开始副本迁移操作，在对应的 Broker 上新建一个 Replica，然后发送 LeaderAndIsr 同步数据，将新的 Replica 加入到 ISR 列表中，如果新的 Replica 加入 ISR 失败，会再次发送 LeaderAndIsr 同步数据，直到新的 Replica 加入到 ISR 中。
* 如果旧的 Replica 为 leader，那么就在新的 ISR 列表中进行 leader 选举。
* 选举完成后下线旧的 Replica，更新 zk 中路径  /admin/reassign_partitions 信息，移除已经成功迁移的 Partition。

### Broker 上线下线

![broker_online_offline](/Users/licheng/Documents/Typora/Picture/broker_online_offline-1594349238785.png)

Controller 会监听 `/Brokers/ids` 这个路径下的所有子节点，如果有新的节点出现，那么就代表有新的 Broker 上线，如果有节点消失，就代表有 Broker 下线。当该节点发生变化时会触发 BrokerChangeListener 的 `HandleChildChange()` 方法。

* 获取 /Brokers/ids 路径下的 Brokerid 列表，与当前 Controller 中保存的 Brokerid 列表比较，找出新增的 BrokerId 和掉线的 Brokerid。

* 对于新上线的 Broker，先在 ControllerChannelManager 中添加该 Broker (即建立与该 Broker 的连接、初始化相应的发送线程和请求队列)，最后 Controller 调用 `onBrokerStartup()` 上线该 Broker；

  * 调用 `sendUpdateMetadataRequest()` 方法向当前集群所有存活的 Broker 发送 Update Metadata 请求，这样的话其他的节点就会知道当前的 Broker 已经上线了；

  * 获取当前节点分配的所有的 Replica 列表，并将其状态转移为 OnlineReplica 状态；

  * 触发 PartitionStateMachine 的 `triggerOnlinePartitionStateChange()` 方法，为所有处于 NewPartition/OfflinePartition 状态的 Partition 进行 leader 选举，如果 leader 选举成功，那么该 Partition 的状态就会转移到 OnlinePartition 状态，否则状态转移失败；

  * 如果副本迁移中有新的 Replica 落在这台新上线的节点上，那么开始执行副本迁移操作。

  * 如果之前由于这个 Topic 设置为删除标志，但是由于其中有 Replica 掉线而导致无法删除，这里在节点启动后，尝试重新执行删除操作。

* 对于掉线的 Broker，先在 ControllerChannelManager 中移除该 Broker (即关闭与 Broker 的连接、关闭相应的发送线程和清空请求队列)，最后 Controller 调用 `onBrokerFailure()` 下线该 Broker。

  * 首先找到 Leader 在该 Broker 上所有 Partition 列表，然后将这些 Partition 的状态全部转移为 OfflinePartition 状态；
  * 触发 PartitionStateMachine 的 `triggerOnlinePartitionStateChange()` 方法，为所有处于 NewPartition/OfflinePartition 状态的 Partition 进行 Leader 选举，如果 Leader 选举成功，那么该 Partition 的状态就会迁移到 OnlinePartition 状态，否则状态转移失败 (掉线时 partition leader 可能在这个 Broker 上，所以要重新选举)；

  * 获取在该 Broker 上的所有 Replica 列表，将其状态转移成 OfflineReplica 状态；

  * 过滤出设置为删除、并且有副本在该节点上的 Topic 列表，先将该 Replica 的转移成 ReplicaDeletionIneligible 状态，然后再将该 Topic 标记为非法删除，即因为有 Replica 掉线导致该 Topic 无法删除；

  * 如果 leader 在该 Broker 上所有 Partition 列表不为空，证明有 Partition 的 leader 需要选举，在最后一步会触发全局 metadata 信息的更新。

### Controller 相关 Zookeeper 路径说明

1. /controller：存放当前 Controller 信息 

   <img src="/Users/licheng/Documents/Typora/Picture/image-20200629181439856.png" alt="image-20200629181439856" style="zoom: 67%;float:left" />

   version：版本号，Brokerid：Controller 对应的 Brokerid，timestamp：竞选成功时的时间戳。

2. /controller_epoch：记录 Controller 变更次数。初试值为 1，每次 Controller 变化，将其值加 1。持久节点。

   <img src="/Users/licheng/Documents/Typora/Picture/image-20200629182754138.png" alt="image-20200629182754138" style="zoom:67%;float:left" />

3. /Brokers/ids/[BrokerId]：存放集群 Broker 信息，包括 ip、端口、jmx 信息

   <img src="/Users/licheng/Documents/Typora/Picture/image-20200629182053915.png" alt="image-20200629182053915" style="zoom:67%;float:left" />

4. /Brokers/topics/[topic]：存放对应 topic 的各个分区当前已分配的副本列表
   <img src="/Users/licheng/Documents/Typora/Picture/image-20200629182233921.png" alt="image-20200629182233921" style="zoom:67%;" />

5. /Brokers/topics/[topic]/partitions/[partitionId]/state：结点存放当前 topic 各个分区的 Leader、ISR、controller_epoch、leader_epoch 等信息。
   <img src="/Users/licheng/Documents/Typora/Picture/image-20200629182333754.png" alt="image-20200629182333754" style="zoom:67%;float:left" />

   

6. /admin：临时结点，只有相关操作时才会存在

   * /admin/delete_topics：要删除的一些topic

   * /admin/reassign_partitions：指导重新分配副本的路径，通过命令修改副本分区时会写入到这个路径下

   * /admin/preferred_replica_election：分区需要重新选举第一个 replica 作为 leader，即 preferred replica。起到自动平衡Leader 分配的作用。

     参数配置：`auto.leader.rebalance.enable=true`