1. **输入，以下是常见得输入内容**
   1) file：从文件系统上的文件读取，与UNIX命令非常相似 tail -0F
   2) syslog：在已知端口上侦听syslog消息进行解析
   3) redis：使用redis通道和redis列表从redis服务器读取。Redis通常用作集中式Logstash安装中的“代理”，该安装将Logstash事件从远程Logstash“托运人”排队。
   4) beats：处理 Beats发送的事件,beats包括filebeat、packetbeat、winlogbeat。
2. **过滤，以下是常见得过滤器**
   1) grok：解析并构造任意文本。Grok是目前Logstash中将非结构化日志数据解析为结构化和可查询内容的最佳方式。Logstash内置了120种模式，您很可能会找到满足您需求的模式！
   2) mutate：对事件字段执行常规转换。您可以重命名，删除，替换和修改事件中的字段。
   3) drop：完全删除事件，例如调试事件。
   4) clone：制作事件的副本，可能添加或删除字段。
   5) geoip：添加有关IP地址的地理位置的信息（也在Kibana中显示惊人的图表！）
3. **输出，以下是常见得输出内容**
   1) elasticsearch：将事件数据发送给Elasticsearch。如果您计划以高效，方便且易于查询的格式保存数据...... Elasticsearch是您的最佳选择
   2) file：将事件数据写入磁盘上的文件。
   3) graphite：将事件数据发送到graphite，这是一种用于存储和绘制指标的流行开源工具。http://graphite.readthedocs.io/en/latest/
   4) statsd：将事件数据发送到statsd，这是一种“侦听统计信息，如计数器和定时器，通过UDP发送并将聚合发送到一个或多个可插入后端服务”的服务。如果您已经在使用statsd，这可能对您有用！

## Logstash 接入 Kafka 的优势

* 可以异步处理数据，防止突发流量。
* 解耦，当 Elasticsearch 异常的时候不会影响上游工作。
* Logstash 过滤消耗资源，如果部署在生产 server 上会影响其性能。

Logstash耗资源较大，运行占用CPU和内存高。另外没有消息队列缓存，存在数据丢失隐患。