#### 配置 hive-env.sh

`cp hive-env.sh.template hive-env.sh`

```shell
# 修改 hive-env.sh，指定 Hadoop 的安装路径
HADOOP_HOME=/home/hduser/hadoop
```

#### 配置 hive-site.xml

新建 hive-site.xml 文件，主要是配置存放元数据的 MySQL 的地址、驱动、用户名和密码等信息

```shell
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop001:3306/hive?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>
    <!--以下配置是为了实现DDL相关操作-->
    <property>
        <name>hive.support.concurrency</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.enforce.bucketing</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.exec.dynamic.partition.mode</name>
        <value>nonstrict</value>
    </property>
    <property>
        <name>hive.txn.manager</name>
        <value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
    </property>
    <property>
        <name>hive.compactor.initiator.on</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.in.test</name>
        <value>true</value>
    </property>
</configuration>

```

#### 拷贝驱动包

将 MySQL 驱动包拷贝到 HIVE_HOME/lib 目录下

#### 初始化元数据

`schematool -dbType mysql -initSchema`

#### 修改 hive 执行引擎

`set hive.execution.engine=spark;`

#### 启动 hive

`hive`