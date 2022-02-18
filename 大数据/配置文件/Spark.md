#### 修改 Spark 配置文件

`cp spark-env.sh.template spark-env.sh`

```shell
# 配置JDK安装位置
JAVA_HOME=/home/hduser/jdk
# 配置hadoop配置文件的位置
HADOOP_CONF_DIR=/home/hduser/hadoop
# 配置zookeeper地址
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=master01:2181,master02:2181,slave01:2181,slave02:2181 -Dspark.deploy.zookeeper.dir=/spark"
```

#### 配置 Worker 节点位置

`cp slaves.template slaves`

```shell
slave01
slave02
slave03
```

#### 启动 Spark

`mv start-all.sh start-spark.sh `

```shell
$SPARK_HOME/bin/start-spark.sh
```

