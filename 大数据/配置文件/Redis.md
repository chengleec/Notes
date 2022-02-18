1. 进入解压后的安装目录

   `make && make install`

2. 修改配置文件

   `vi redis.conf`

   ```shell
   bind 192.168.5.101 127.0.0.1 # 绑定端口
   daemonize yes   # Redis后台运行
   requirepass 123456   #指定Redis的密码
   dir /home/hduser/redis/logs   #Redis数据存储的位置
   appendonly yes   # 开启aof日志，它会每次写操作记录一条日志
   ```

3. 启动服务

   `redis-server redis/redis.conf`

4. 连接

   ```shell
   redis-cli -h [hostname] -p [port]  #port默认为6397
   ```