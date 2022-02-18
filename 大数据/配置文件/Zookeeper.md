#### 修改配置文件

   ```shell
cp zoo_sample.cfg  zoo.cfg
    # The number of milliseconds of each tick（用于计算的最小时间单位）
    tickTime=2000
    # The number of ticks that the initial synchronization phase can take
    initLimit=10
    # The number of ticks that can pass between sending a request and getting an acknowledgement（心跳机制）
    syncLimit=5
    # the directory where the snapshot is stored.do not use /tmp for storage, /tmp here is just example sakes.
	# 指定数据存储目录				
	dataDir=/usr/local/zookeeper/data
	# 指定日志文件目录
    dataLogDir=/usr/local/zookeeper/log
    # the port at which the clients will connect
    clientPort=2181
    # the maximum number of client connections.increase this if you need to handle more clients
    # maxClientCnxns=60
    # 指名集群间通讯端口和选举端口
    server.1=hadoop001:2287:3387
    server.2=hadoop002:2287:3387
    server.3=hadoop003:2287:3387
   ```

#### 新建myid文件

```shell
# hadoop001主机
echo "1" > $dataDir/myid
# hadoop002主机
echo "2" > $dataDir/myid
# hadoop003主机
echo "3" > $dataDir/myid
```

#### 启动zookeeper并验证

```shell
[root@hadoop001 bin] zkServer.sh start
[root@hadoop001 bin] jps # 使用 JPS 验证进程是否已经启动，出现 QuorumPeerMain 则代表启动成功。
3814 QuorumPeerMain
```

#### 编写zookeeper集群脚本
```shell
# 创建启动脚本
touch start-zookeeper.sh
# 赋予执行权限
chmod 777 start-zookeeper.sh
vi start-zookeeper
    #!/bin/sh
    for host in hadoop001 hadoop002 hadoop003
    do
        echo "$host starting zookeeper  ……"
        ssh $host "source /etc/profile;zkServer.sh start"
    done
```

```shell
#创建停止脚本
touch stop-zookeeper.sh
# 赋予执行权限
chmod 777 start-zookeeper.sh
vi start-zookeeper
    #!/bin/sh
    for host in hadoop001 hadoop002 hadoop003
    do
        echo "$host stopping zookeeper  ……"
        ssh $host "source /etc/profile;zkServer.sh stop"
    done
```

​    