#### 集群规划

| 主机名   | IP            | 软件      | 进程                                                         |
| -------- | ------------- | --------- | ------------------------------------------------------------ |
| master01 | 192.168.5.101 | zookeeper | NameNode,ResourceManager,JournalNode,QuorumPeerMain,DfszkFailoverController |
| master02 | 192.168.5.102 | zookeeper | NameNode,ResourceManager,JournalNode,QuorumPeerMainDfszkFailoverController |
| slave01  | 192.168.5.201 | zookeeper | DataNode,NodeManager,JournalNode,QuorumPeerMain              |
| slave02  | 192.168.5.202 | zookeeper | DataNode,NodeManager,JournalNode,QuorumPeerMain              |
| slave03  | 192.168.5.203 | zookeeper | DataNode,NodeManager,JournalNode,QuorumPeerMain              |

#### 配置 core-site.xml

`vi $HADOOP_HOME/etc/hadoop/core-site.xml`

```shell
<configuration>
    <property>
        <!--指定 namenode 的 hdfs 协议文件系统的通信地址-->
        <name>fs.defaultFS</name>
        <value>hdfs://ns</value>
    </property>
    <property>
        <!--指定 hadoop 集群存储临时文件的目录-->
        <name>hadoop.tmp.dir</name>
        <value>/home/hduser/hadoop-3.2.1/tmp</value>
    </property>
    <property>
        <!--指定zookeeper地址-->
        <name>ha.zookeeper.quorum</name>
        <value>master01:2181,master02:2181,slave01:2181,slave02:2181,slave03:2181</value>
    </property>
    <!--指定 IPC 重试次数和间隔，防止出现 NameNode 连接 journalNode 超时的情况-->
    <property>
        <name>ipc.client.connect.max.retries</name>
        <value>100</value>
    </property>
    <property>
        <name>ipc.client.connect.retry.interval</name>
        <value>5000</value>
    </property>
</configuration>
```

#### 配置 hdfs-site.xml

`vi $HADOOP_HOME/etc/hadoop/hdfs-site.xml`

```shell
<configuration>
    <property>
        <!-- 集群服务的逻辑名称 -->
        <name>dfs.nameservices</name>
        <value>ns</value>
    </property>
    <property>
        <!-- NameNode ID 列表-->
        <name>dfs.ha.namenodes.ns</name>
        <value>nn1,nn2</value>
    </property>
    <property>
        <!-- nn1 的 RPC 通信地址 -->
        <name>dfs.namenode.rpc-address.ns.nn1</name>
        <value>master01:9000</value>
    </property>
    <property>
        <!-- nn2 的 RPC 通信地址 -->
        <name>dfs.namenode.rpc-address.ns.nn2</name>
        <value>master02:9000</value>
    </property>
    <property>
        <!-- nn1 的 http 通信地址 -->
        <name>dfs.namenode.http-address.ns.nn1</name>
        <value>master01:50070</value>
    </property>
    <property>
        <!-- nn2 的 http 通信地址 -->
        <name>dfs.namenode.http-address.ns.nn2</name>
        <value>master02:50070</value>
    </property>
    <property>
        <!-- NameNode 元数据在 JournalNode 上的共享存储目录 -->
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://master01:8485;master02:8485;slave01:8485;slave02:8485;slave03:8485/ns</value>
    </property>
    <property>
        <!-- Journal Edit Files 的存储目录 -->
        <name>dfs.journalnode.edits.dir</name>
        <value>/home/hduser/hadoop-3.2.1/journal/data</value>
    </property>
    <property>
        <!-- 访问代理类，用于确定当前处于 Active 状态的 NameNode -->
        <name>dfs.client.failover.proxy.provider.ns</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <!-- 开启故障自动转移 -->
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <!-- 配置隔离机制，确保在任何给定时间只有一个 NameNode 处于活动状态 -->
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>`
    </property>
    <property>
        <!-- 使用 sshfence 机制时需要 ssh 免密登录 -->
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/hduser/.ssh/id_rsa</value>
    </property>
    <property>
        <!-- SSH 超时时间 -->
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>30000</value>
    </property>
    <property>
        <!--namenode 节点数据（即元数据）的存放位置，可以指定多个目录实现容错，多个目录用逗号分隔-->
        <name>dfs.namenode.name.dir</name>
        <value>/home/hduser/hadoop-3.2.1/namenode/data</value>
    </property>
    <property>
        <!--datanode 节点数据（即数据块）的存放位置-->
        <name>dfs.datanode.data.dir</name>
        <value>/home/hduser/hadoop-3.2.1/datanode/data</value>
    </property>
    <property>
    	<!--副本数量-->
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

#### 配置 mapred-site.xml

`vi  $HADOOP_HOME/etc/hadoop/mapred-site.xml`

```shell
<configuration>
    <property>
        <!--指定 mapreduce 作业运行在 yarn 上-->
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

#### 配置 yarn-site.xml

` vi $HADOOP_HOME/etc/hadoop/yarn-site.xml`

```shell
<configuration>
    <property>
    	<!-- 开启RM高可用 -->
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <property>
    	<!-- 指定RM的cluster id -->
        <name>yarn.resourcemanager.cluster-id</name>
        <value>yarn-ha</value>
    </property>
    <property>
    	<!-- 指定RM的名字 -->
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <property>
    	<!-- 分别指定RM的地址 -->
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>master01</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>master02</value>
    </property>
    <property>
    	<!-- 指定zk集群地址hadoop-2.8.5 -->
        <name>yarn.resourcemanager.zk-address</name>
        <value>slave01:2181,slave02:2181,slave03:2181</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

#### 配置slaves

`vi $HADOOP_HOME/etc/hadoop/slaves`

```shell
slave01
slave02
slave03
```

#### 启动 Hadoop-HA

```shell
# 启动zookeeper集群
$ZOOKEEPER_HOME/bin/zkServer.sh start
# 查看zookeeper状态
$ZOOKEEPER_HOME/bin/zkServer.sh status
# 格式化 ZooKeeper 集群
$ hdfs zkfc -formatZK
# 启动 JournalNode 集群分别在 slave01、slave02、slave03 上执行以下命令
$ hadoop-daemon.sh start journalnode
# 格式化集群的 NameNode (在master01上执行)
$ hdfs namenode -format
# 启动刚格式化的 NameNode (在master01上执行)
$ hadoop-daemon.sh start namenode
# 同步 NameNode1 元数据到 NameNode2 上 (在master02上执行)  
$ hdfs namenode -bootstrapStandby 或者 scp -r data master02:/home/hduser/hadoop-3.2.1/namenode/data（将元数据文件夹拷贝到standby节点）
# 启动 NameNode2 (在master02上执行)
$ hadoop-daemon.sh start namenode
# 启动集群中所有的DataNode (在master01上执行)  
$ sbin/start-dfs.sh
# 验证ha(在master01节点停掉namenode进程)
$ hadoop-daemon.sh stop namenode
```

#### 启动 yarn-HA

```shell
# 在 RM1 启动 YARN (在master01上执行)
$ start-yarn.sh
# 在 RM2 启动 YARN (在master02上执行)
$ yarn-daemon.sh start resourcemanager
# 在任意节点执行获取resourceManager状态（active）
$ yarn rmadmin -getServiceState rm1
# 在任意节点执行获取resourceManager状态（standby）
$ yarn rmadmin -getServiceState rm2
# 验证 yarn 的 ha（在master01节点执行）standby 的 resourcemanager 则会转换为 active
$ yarn-daemon.sh stop resourcemanager
# 在任意节点执行获取resourceManager状态（active）
$ yarn rmadmin -getServiceState rm2
```

