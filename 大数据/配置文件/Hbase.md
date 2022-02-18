#### 配置 hbase-env.sh

```shell
# The java implementation to use.  Java 1.8+ required.
export JAVA_HOME=/home/hduser/jdk1.8.0_231
# Tell HBase whether it should manage it's own instance of ZooKeeper or not.
export HBASE_MANAGES_ZK=false
```

#### 配置hbase-site.xml

```shell
<configuration>
    <property>
        <!-- 指定 hbase 以分布式集群的方式运行 -->
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <!-- 指定 hbase 在 HDFS 上的存储位置 -->
        <name>hbase.rootdir</name>
        <value>hdfs://ns/hbase</value>
    </property>
    <property>
        <!-- 指定 zookeeper 的地址-->
        <name>hbase.zookeeper.quorum</name>
        <value>master01:2181,master02:2181,slave01:2181,slave02:2181,slave03:2181</value>
    </property>
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>false</value>
    </property>
</configuration>
```

#### 配置 regionserver、backup-master地址

`vi regionserver`

```shell
slave01
slave02
slave03
```

`vi backup-masters`

`master02`

#### 拷贝文件

将hadoop目录中的core-site.xml hdfs-site.xml拷贝到hbase的conf目录中

#### web-UI

16010

#### 集成 Phoenix

```shell
cp $PHOENIX_HOME/phoenix-5.0.0-HBase-2.0-client.jar phoenix-core-5.0.0-HBase-2.0.jar $HBASE_HOME/lib 
cp $HBASE_HOME/conf/hbase-site.xml $PHOENIX_HOME/bin
cp $HADOOP_HOME/etc/hadoop/core-site.xml hdfs-site.xml $PHOENIX_HOME/bin
```