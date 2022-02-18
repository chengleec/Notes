#### 修改配置文件

```shell
# The id of the broker. 集群中每个节点的唯一标识
broker.id=0
# 监听地址
listeners=PLAINTEXT://:9092
# 数据的存储位置
log.dirs=/usr/local/kafka-logs
# Zookeeper连接地址
zookeeper.connect=hadoop001:2181,hadoop001:2182,hadoop001:2183
```

