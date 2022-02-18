### 一、项目流程图

![大数据平台部 > 指标预热项目整理 > image2021-10-15_17-16-28.png](http://wiki.lianjia.com/download/attachments/947540776/image2021-10-15_17-16-28.png?version=1&modificationDate=1634289388526&api=v2)

### 二、项目部署

1. 将 jar 包拷贝到 /home/druid/pre_heat/ 目录下，并重命名为 preheat-0.0.1-SNAPSHOT-stable.jar，运行 stop_preheat.sh 脚本，停止之前运行的 preheat 任务，运行 start_preheat.sh，启动最新的 preheat 任务。
2.  /home/druid/pre_heat/ 目录下还包含三个脚本文件，preheat_cockpit：预热 cockpit 数据，preheat_ljzc：预热 ljzc 数据源，preheat_roles_databoard：预热 roles_databoard 数据源
3. 任务日志路径为 /home/druid/pre_heat/#{date}

#### 测试项目部署

测试项目路径为 /home/druid/pre_heat_test/目录，jar 包名称为 preheat-hystrix，已修改，并可以复现 hystrix 问题，测试端口号为 8090，其他与线上项目部署一致。

测试代码分支为 dev_licheng_1015_hystrix，地址：https://git.lianjia.com/data-webdev/measure-data-center/-/tree/dev_licheng_1015_hystrix/





![image-20211227172944284](/Users/licheng/Documents/Typora/其他/Untitled 2.assets/image-20211227172944284.png)