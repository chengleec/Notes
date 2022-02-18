#### 修改 masters

`vi $FLINK_HOME/conf/masters`

```shell
master01:8081
master02:8081
```

#### 修改 slaves

`vi $FLINK_HOME/conf/slaves`

```shell
slave01
slave02
slave03
```

#### 修改 flink-conf.yaml

`vi $FLINK_HOME/conf/flink-conf.yaml`

```shell
jobmanager.rpc.address: master01,master02
# 开启高可用模式
high-availability: zookeeper
# 指定 HDFS 的路径，用于存储 JobManager 的元数据
high-availability.storageDir: hdfs://ns/flink/ha/
# 配置 zk 各个节点的端口
high-availability.zookeeper.quorum: master01:2181,master01:2181,master02:2181,slave01:2181,slave02:2181,slave03:2181
# zk 节点根目录，放置所有 flink 集群节点的 namespace
high-availability.zookeeper.path.root: /flink
# zk 节点集群 id，放置了 flink 集群所需要的所有协调数据
high-availability.cluster-id: /cluster_one
# 配置Yarn重试次数
yarn.application-attempts: 10
```

#### 修改 yarn-site.xml

```shell
<!-- 设置提交应用程序的最大尝试次数 -->
<property>
    <name>yarn.resourcemanager.am.max-attempts</name>
    <value>4</value>
</property>
```

#### 启动 flink 集群

`start-cluster.sh`

#### WebUI

8081