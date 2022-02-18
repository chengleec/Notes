#### 背景

1. HiveToDruid 任务介绍

   Druid 是一个分布式的、支持实时多维 OLAP 分析的数据处理系统。它既支持高速的数据实时摄入处理，也支持实时且灵活的多维数据分析查询。因此 Druid 最常用的场景就是大数据背景下、灵活快速的多维 OLAP 分析。 另外，Druid 还有一个关键的特点：它支持根据时间戳对数据进行预聚合摄入和聚合分析，因此在有时序数据处理分析的场景中会经常用到它。

   Druid 支持流式摄取（实时）和批量摄取（离线）两种数据导入方式，从文件进行批加载时有三个选项：index_parallel（本地并行批任务）、index_hadoop（基于hadoop）或 index（本地简单批任务）。Druid 支持的导入数据格式有  JSON、CSV、ORC 和 Parquet。所以在进行 HiveToDruid 数据导入时，需要先将 Hive 表数据转化为 Parquet 文件，再执行 index_hadoop 任务。

   Hive 表数据导入 Druid 主要是通过 Hadoop Index 服务实现，Hadoop_Index 主要分为两个过程，分别由DeterminePartitionsJob 和 IndexGeneratorJob 完成。

   DeterminePartitionsJob 的工作主要是使用 HyperLogLog 来估算数据集的基数，以此决定一个合适的分片数量，每个分片有一个 reduce 去执行，这样使得每个 reduce 操作数据不至于过大的同时提高并发度，提升数据导入效率。

   确定 partition 有 dynamic 、single_dim 和 hash 三种方式。dynamic 默认每 500 万行生成一个 partition；single_dim 会按照单一维度进行分区，但最好这个维度拥有非常高的基数，否则会造成数据不均衡，影响查询效率；最常用的为 hash 模式，它会按照所有维度作 hash 分区，能够相对均衡的将所有数据平均分配，详细信息可以查看它的实现类 DetermineHashedPartitoinsJob。

   IndexGeneratorJob 的工作主要是对原始数据进行解析和过滤成符合要求的 InputRow，然后按照 InputRow 的时间进行分桶，最后根据 DetermineHashedPartitionsJob 估算的分片数量创建 segment，导入数据。

2. 为什么 HiveToDruid 任务要使用 shell 脚本来运行

   一般的 HiveToDruid 任务会在 vili 平台通过 Cube 建模构建，构建流程为通过 Model 定义事实表和维度表的 join 关系，指定维度字段和度量字段。Cube 是在 Model 层上再选取，选取需要作为指标查询的维度和度量，同时指定度量规则，Model 和 Cube 的关系可以是一对多。Cube 建立完成后，平台针对离线 Cube 会帮助用户自动配置例行调度作业，可进行小时级、天级、周级、月级、年级周期配置。

   但某些时间表达式比较复杂的任务无法直接在 vili 上调度，例如我所负责的该 HiveToDruid 任务需要每天将该月所有的数据导入到该月 1 号的 pt 中，且如果当天为 1 号，需要将数据导入到上个月 1 号的 pt 中，对于这种时间表达式比较复杂的任务需要单独编写 shell 脚本来运行。

3. 为什么 shell 任务要迁移到 docker 中

   * 统一收拢线上提交客户端后，可以方便与安全的对集群进行联合安全审计；
   * 调度节点都在调度和集群侧，集群如果再有类似集群搬迁，多机房建设，用户无需关心这类任务的搬迁；
   * 集群进行优化代码上线和配置变更时候，直接进行对应调度机器的修改，无需用户侧介入；
   * DockerShell 可以避免新人 shell 误操作，造成不可挽回的损失。

#### 数据导入过程

1. 数据导入架构图

   <img src="http://r1vidgj7p.hn-bkt.clouddn.com/image/1image-20211010105634100.png" alt="image-20211012105307229.png" style="zoom:67%;" />

2. 数据导入过程：

   ① 通过 Spark Sql 从 hive 数仓中获取数据并生成 parquet 的文件，parquet 文件就绪后会通知 Druid overload（执行数据灌入的 master 节点）数据已经就绪。

   ② overload 会去加载 hdfs 上的 parquet 文件，开始执行 hadoop index 作业。Hadoop index 作业的执行主要是分为三个步骤：

   * 第一步是 partition Job，这一步决定会分多少个 segments。
   * 第二步是构建字典。
   * 第三步是生成索引作业，针对维度列和度量列生成倒排索引和 bitmap 索引。

   ③ Hadoop index 作业运行完后，segment 持久化到深度存储 hdfs，落盘后 historical 从 hdfs 上拉取文件，生成自有存储格式，这样整个数据导入就结束了。

3. 离线数据导入加速

   <img src="http://r1vidgj7p.hn-bkt.clouddn.com/image/1image-20211011165050883.png" alt="image-20211011165050883.png" style="zoom:67%;" />

在生产环境中，如果分区 segment 较大，数据不会均衡的分布到各个 historical 节点上，导致查询效率较低，因此一般会对分区数据进一步切分，使之数据分布更加均衡。

对于每日分区数据量比较大的增量 Hive 表，在导入数据时候我们会按照数据行数指定合理的 numShards 数量，通过指定 numShards 数量，我们可以增加 Reduce 容器个数，提升 IndexGeneratorJob 导入速度，同时因为指定了 numShards 数量和 intervals 信息，我们可以跳过 DeterminePartitionsJob 过程，进一步加速 Hadoop_Index 过程。

最后，我们通过 Spark Sql 重新 repartition 分区调整生成 parquet 文件的数量，从而增加数据摄入阶段 Map 容器的个数，通过这些举措提升数据源导入的速度。

关键代码：

```java 
if (rowNum > maxRowNumPerPt) {
    // 确定 parquet 文件数量
    Double num = Math.ceil(rowNum.doubleValue() / maxRowNumPerPt.doubleValue());
    Integer repartitionNum = num.intValue();
    // 确定 numShard 数量
    Double shardNumDouble = Math.ceil(repartitionNum.doubleValue() / shardsPerPt.doubleValue());
    // repartition 并创建临时视图
    String tempView = olapTableName.split("\\.")[1] + "_tmp_view";
    olapDataIn.repartition(repartitionNum).createOrReplaceTempView(tempView);
    Map map = tjb.getExtraProperties();
}
```

#### 容器化过程

在调度系统创建一个docker-shell 任务，按步骤操作即可，创建地址：http://diaodu.data.ke.com/task-create

1. 申请 Hadoop 用户，获取调度任务权限。

   > 在申请用户的过程中遇到了一个问题，我需要转换的 shell 脚本为 HiveToDruid 任务，需要同时获取读取 hive 表权限和写入 druid 的权限，但创建 DockerShell 任务时只能指定一个用户，无法满足我的需求。最后的解决方案为使用 druid 用户，将 shell 脚本需要用到的 4 张 hive 表的读权限赋予该 druid 用户。希望后续能对该问题有所优化。

2. 根据 shell 脚本所依赖的环境选择相应的 docker 镜像。

3. 上传所需文件，包括 shell 脚本，需要用到的 jar 包等等。

   > jar 包修改注意事项：
   >
   > * 在 jar 包中不需要指定 hive-site.xml 路径，同样不需要 kerberos 认证，这些内容系统会自动填充。
   > * 如果需要指定路径，应该使用相对路径，例如 spark 的 bin 目录可以使用`System.getenv("SPARK_HOME") + “/bin”` 拼接获得。
   > * 如果存在读写文件，需要注意用户权限问题。

4. 填写启动命令。脚本启动的入口，启动命令与 shell 任务的启动命令相同，只需要注意路径问题即可。

##### 容器化过程释疑

1. 为什么不需要 kerberos 认证

   镜像里并没有配置 Kerberos，docker 运行时会通过挂载的方式将宿主机的 keytab 文件加载到容器的 /tmp 目录下，当需要 kerberos 认证时，程序会读取 /tmp 目录内容完成 kerberos 认证过程。

2. 程序的运行日志是怎么收集的

   因为容器运行结束后注销，所以运行容器的程序会捕捉 shell 脚本的打印日志和错误日志，并保存到宿主机，方便随时查看。

#### 总结

本篇文章首先介绍了 HiveToDruid 数据导入的过程，并指出了部分业务需求的特殊性和 shell 任务转换为 docker-shell 任务的必要性，之后介绍了在自己的实践过程中对 druid 离线数据导入过程做的一部分优化，与容器化过程中遇到的问题与解决方案，还有一部分容器化的过程的注意事项，希望能对读者有一些帮助。

相比物理部署，docker 任务可以更轻松的实现迁移和扩展，更高效的利用系统资源，且在安装好 docker 环境物理机上，可以直接从仓库拉取镜像，轻松地提高任务并行度。

#### 参考

https://druid.apache.org/docs/latest/design/

https://github.com/apache/druid/blob/8296123d895db7d06bc4517db5e767afb7862b83/indexing-hadoop/src/main/java/org/apache/druid/indexer/DetermineHashedPartitionsJob.java

https://blog.csdn.net/gg782112163/article/details/70211615

