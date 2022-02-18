### MapReduce 介绍

MapReduce 是一个用于大规模数据处理的分布式计算框架，用户需要自己定义 Map 函数和 Reduce 函数，Map函数用来处理原始数据( 初始键值对) 以生成一批中间的 key/value 对，Reduce 函数将所有这些中间的有着相同key的values合并起来。

### MapReduce 运行流程

<img src="/Users/licheng/Documents/Typora/Picture/Yarn工作流程.png" style="zoom:67%;" />

1. 首先从` job.waitForCompletion()`方法开始，向集群提交 MapReduce 作业 。

2. Resource Manager 向客户端返回一个作业 id 和资源提交路径 。

3. 客户端将 job 所需资源 (包括 Jar 包，配置文件, split 信息) 提交到指定路径（HDFS）。然后调用 `submitApplication() `来提交 job。

4. Resource Manager 通过调度器 (scheduler) 在 NodeManager 创建一个 container，然后在该 container 内启动 AppMaster 进程。

5. 然后 AppMaster 从提交的资源中获取分片信息，向 ResourceManager 申请资源，为每个输入切片创建一个 MapTask,  创建指定数量的 ReduceTask。

6. 利用指定的 InputFormat 来获取 RecordReader 对象从 HDFS 读取数据，形成 kv 输入。执行用户定义的 Map 方法，产生 kv 输出。

7. Map 输出数据会写入到一个环形缓冲区中，当达到阈值 0.8 时，会先按照 Partition 分区 (hashPartitoner)、再按照 key 进行排序 (quick sort)，如果有 combiner，也会执行 combiner，最后将结果溢写到磁盘。

   > 缓冲区的作用是批量收集 Map 结果，减少磁盘 IO 的影响。

8. 如果溢写文件个数大于 3 时，会执行 Merge 操作，即将多个溢写文件合并为一个大文件 (归并排序)，会再执行一次 combiner 操作。

9. Reduce 端默认有 5 个数据复制线程会从 Map 端复制数据到 Reduce 端。首先将数据写到 Reduce 端的缓存中，同样缓存占用到达一定阈值后会将数据写到磁盘中。如果形成了多个磁盘文件还会进行合并 (归并排序)，最后一次合并的结果作为 Reduce 的输入而不是写入到磁盘中。

   > 有一个 Map 完成就可以开始复制了。

10. 调用用户定义的 Reduce 方法进行逻辑运算，收集输出的结果，调用用户指定的 OutputFormat 将结果数据输出到 HDFS。
11. 当所有 Map/Reduce Task 都完成时，ApplicationMaster 向 ResourceManager 汇报任务完成，并注销自己。

>shuffle过程
>
><center class="half">
><img src="C:\Users\admin\Typora\Picture\MapReduce流程Map端.png" style="zoom:50%;"/><img src="C:\Users\admin\Typora\Picture\MapReduce流程Reduce端.png" style="zoom:50%;" />
></center>
>
>
>

### MapReduce Task 数目划分

#### MapTask

MapTask 数量由输入文件的数目、输入文件的大小和配置参数决定的。

* 对于每一个输入的文件会有一个 InputSplit，每一个 InputSplit 会启动一个 MapTask 运行。

* 如果输入文件超过了 HDFS 块的大小 (128M)，那么对于同一个输入文件会有多个的 MapTask 运行起来。

>会有一个比例进行运算来进行切片，为了减少资源的浪费。
>例如一个文件大小为 260 M，在进行 MapReduce 运算时，会首先用 260M/128M，得出的结果和 1.1 进行比较。大于则切分出一个 128M 作为一个分片，剩余 132M，再次除以 128，得到结果为 1.03，小于1.1，则将 132 作为一个切片，即最终 260M 被切分为两个切片进行处理，而非 3 个切片。

#### ReduceTask

Reduce 任务是一个数据聚合的步骤，数量默认为 1，一个 Reduce 任务对应一个输出文件。

自定义 ReduceTask 个数：

* 参数设置：`MapReduce.job.Reduces`
* 编程指定：`job.setNumReduceTasks()`

### MapReduce 优化

* 调整 Map 端环形缓冲区大小，默认 100 M，可以适当增大，调整到 512 M，减少溢写文件数量。

  相关参数：

  * `mapreduce.task.io.sort.mb`：MapTask 缓冲区所占内存大小，默认 100 M。

  * `mapreduce.map.sort.spill.percent`，MapTask 溢写阈值，默认 0.8。

* 调整 Merge 过程合并文件数量，默认最多为 10 个，可以适当增大，调整到 64 个，减少合并的次数，降低对磁盘操作的次数。

  相关参数：``mapreduce.task.io.sort.factor`：每次 Merge 操作合并的文件数量。Map 端和 Reduce 端都是通过此参数控制。

* 添加 Combine 过程，减少溢写文件数量，提高磁盘 IO 性能。

* 对 Map 端的输出数据进行压缩、指定压缩方式。

  相关参数： `mapreduce.map.output.compress` ：Map 输出是否进行压缩，默认为 false。如果压缩就会多耗 CPU，但是减少传输时间，如果不压缩，就需要较多的传输带宽。

* Reduce 端复制数据的线程数默认为 5，如果 MapTask 数量很多的话，可以适当调大。

  相关参数：`mapreduce.reduce.shuffle.parallelcopies`：Reduce 端复制线程数量。

* 调整 Reduce 端内存缓冲区的大小，Reduce 端内存大小与 JVM 堆内存大小相关，默认为堆内存大小的 70%。

  相关参数：

  * `mapreduce.reduce.shuffle.input.buffer.percent`：占堆内存大小的比例，默认 70%。
  * `mapreduce.reduce.shuffle.merge.percent`：Reduce 端溢写文件的阈值，默认 0.66。

* 如果不是做聚合类的运算，可以启用多个 ReduceTask，提高并行度，加快执行速度。