#### 拷贝文件

拷贝 HBase 的包和 hive 包到 spark 的 jars 目录下

* hbase-client..
* hbase-common..
* hbase-protocol..
* hbase-server..
* hive-hbase-handler..
* htrace-core..
* mysql-connector-java-..

#### 启动spark-shell

实质是 Spark Sql 通过 hive 外部表来获取 HBase 的表数据

`bin/spark-shell`

```scala
val df =spark.sql("select count(1) from weblogs").show
```

