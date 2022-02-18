#### 配置文件

* 将hive的配置文件 hive-site.xml 拷贝到 spark conf 目录，同时添加 metastore 的url配置。

  ```shell
  <property>
      <name>hive.metastore.uris</name>
      <value>thrift://master01:9083</value>  # 9083是默认端口
  </property>
  ```

* 拷贝 hive 中的 mysql jar 包到 spark 的 jar 目录下

  `cp hive-0.13.1-bin/lib/mysql-connector-java-5.1.27-bin.jar spark-2.2-bin/jars/`

* 检查 spark-env.sh 文件中的配置项

  `vi spark-env.sh`

  `HADOOP_CONF_DIR=/home/hduser/hadoop-3.2.1/etc/hadoop`

#### 启动服务

* 检查 mysql 是否启动

  ```shell
  # 查看 mysql 启动状态
  service mysqld status
  # 启动 mysql 服务
  service mysqld start
  ```

* 启动 hive metastore 服务

  `bin/hive --service metastore`

* 启动 spark-shell，使用 spark 访问 hive 中的数据表

  `bin/spark-shell`

  ```scala
  # spark-hive 为数据库名，test 为数据表名
  spark.sql("select * from spark-hive.test").show
  ```

  

