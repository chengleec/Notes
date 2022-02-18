#### 修改 es.yml

`vi $ES_HOME/config/elasticsearch.yml`

```shell
#集群的名称  
#日志文件会以集群名称命名
cluster.name: ESCluster  
#当前机器节点名称  
node.name: master01  

#索引数据的存储路径  
path.data: /usr/local/elasticsearch/data  
#日志文件的存储路径  
path.logs: /usr/local/elasticsearch/logs  

#设置为true来锁住内存。因为内存交换到磁盘对服务器性能来说是致命的，当jvm开始swapping时es的效率会降低，所以要保证它不swap  
bootstrap.memory_lock: true  
bootstrap.system_call_filter: false

#绑定的ip地址  
network.host: 0.0.0.0  
#设置对外服务的http端口，默认为9200  
http.port: 9200  

# 集群节点
discovery.seed_hosts: ["master01","master02","slave01","slave02","slave03"]
# 可以作为 master 的节点
cluster.initial_master_nodes: ["master01","master02","slave01","slave02","slave03"]

```

#### 调整 jvm 内存

`vi $ES_HOME/conf/jvm.options`

```shell
# 官方建议为物理内存的一半
-Xms512m  
-Xmx512m
```

#### 修改 limits.conf

`vi /etc/security/limits.conf` （以 root 身份登录）

```shell
# 报错信息 max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
hduser hard nofile 65536
hduser soft nofile 65536
# 报错信息 max number of threads [1024] for user [hduser] is too low, increase to at least [4096]
hduser soft nproc 4096
hduser hard nproc 4096

hduser soft memlock unlimited
hduser hard memlock unlimited
```

#### 修改 sysctl.conf

`vi /etc/sysctl.conf` (以 root 身份登录)

```shell
vm.max_map_count=655360
```

#### 启动

```shell
# 以后台方式启动 ES
elasticsearch -d 
# 以后台方式启动 ElasticHD
ElasticHD -p master01:9800 &
```

#### 查看

```shell
master01:9200
```

