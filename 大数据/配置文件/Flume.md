#### 修改配置文件

```shell
mv flume-env.sh.template flume-env.sh
# 指定 JDk 安装路径
export JAVA_HOME=/usr/java/jdk1.8.0_201
```

#### 验证

`flume-ng version`，出现版本号，代表安装成功