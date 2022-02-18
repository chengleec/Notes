### 概述

在 Spark 1.2 以前，默认采用的是 HashShuffle。这种 Shuffle 机制会产生大量的中间磁盘文件，进而由大量的磁盘 IO 操作影响了性能。因此在 Spark 1.2 以后的版本中，默认的 Shuffle 机制改成了 SortShuffle。

SortShuffle 相较于 HashShuffle，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并成一个磁盘文件。在 ShuffleRead 时，只要根据索引读取每个磁盘文件中的部分数据即可。

### HashShuffle 机制

#### 未优化的 HashShuffle

<img src="/Users/licheng/Documents/Typora/Picture/image-20200805095206904.png" alt="image-20200805095206904" style="zoom: 50%;" />

在 HashShuffle 没有优化之前，每一个 ShufflleMapTask 会为下游每一个 Task 创建一个 bucket 缓存，final RDD 每输出一个 record 就根据其 hash partitioner 的结果放入对应的 bucket 缓冲区中。当数据达到一定阈值后将 bucket 中的数据刷新到磁盘上，即对应的 ShuffleBlockFile。

相关参数：`spark.shuffle.file.buffer.kb` ，缓冲区大小，默认是 32KB。

然后 ShuffleMapTask 将输出结果作为 MapStatus 发送到 DAGScheduler 的 MapOutputTrackerMaster，每一个 MapStatus 包含了输出数据的位置和大小。

ReduceTask 然后去利用 BlockStoreShuffleFetcher 向 MapOutputTrackerMaster 获取 MapStatus，看哪一份数据是属于自己的，然后底层通过 BlockManager 将数据拉取过来。

**拉取过来的数据会组成一个内部的 ShuffleRDD，优先放入内存，内存不够用则放入磁盘，然后 ReduceTask 开始进行聚合，最后生成我们希望获取的那个MapPartitionRDD**

**缺点**

* 产生的文件过多：每个 ShuffleMapTask 产生 ReduceTask 个 ShuffleBlockFile，M 个 ShuffleMapTask 就会产生 M * R 个文件，磁盘上会存在大量的数据文件。
* 缓冲区占用内存空间大。M 个 ShuffleMapTask 就会产生 M * R 个缓冲区，有可能造成内存溢出。

#### 优化后的 HashShuffle

<img src="/Users/licheng/Documents/Typora/Picture/image-20200805101501079.png" alt="image-20200805101501079" style="zoom:50%;" />

在一个 core 上连续执行的 ShuffleMapTasks 可以复用 bucket 缓冲区和 ShuffleBlockFile 输出文件。先执行完的 ShuffleMapTask 形成 ShuffleBlockFile，后执行的 ShuffleMapTask 可以将输出数据直接追加到 ShuffleFile 后面。下一个 Stage 的 Task 只需要 fetch 整个 ShuffleBlockFile 就行了。这样每个 Worker 持有的文件数降为 cores * R。

**缺点：**

如果 Reduce 端的并行任务或者是数据分片过多的话依旧会产生很多小文件。

### Sort-Based Shuffle 机制

该机制每一个ShuffleMapTask不会为后续的任务创建单独的文件，而是会将所有的 Task 结果写入同一个文件，并且生成一个索引文件。

Sort-Based Shuffle有几种不同的策略：BypassMergeSortShuffleWriter、SortShuffleWriter 和 UnasfeSortShuffleWriter。

#### BypassMergeSortShuffleWriter

与 HashShuffle 基本一致，唯一的区别在于 ShuffleMapTask 的输出文件会合并成同一个文件，同时生成一个索引文件，标记每个分区的偏移量。

该模式主要用于不需要排序和聚合的 Shuffle 操作，并且 ReduceTask 数量较少的情况。

相关参数：`spark.shuffle.sort.bypassMergeThrshold`，默认值 200。ReduceTask 数量小于这个参数时，会采用这种模式。

优点：不需要排序，可以节省这部分性能开销。

#### SortShuffleWriter

* 首先会判断 Map 端是否需要聚合操作，如果需要聚合 (reducebykey)，则采用 PartitionedAppendOnlyMap；如果不需要聚合 (groupby)，则采用 PartitionedPairBuffer 将数据添加到内存中并排序。

  > SortShuffleWriter 使用 ExternalSorter 进行排序，首先根据 partition id 排序，如果 partition id 相同，再根据 hashcode 排序。
  >
  > PartitionedAppendOnlyMap 自己实现了哈希表，采用了二次探测算法避免哈希冲突。

* 然后当内存空间不足时，就将内存中的数据溢写到磁盘。

* 最后将磁盘中所有的临时文件通过归并排序生成一个 Data 文件和一个索引文件。

### Spark 与 MapReduce Shuffle 的异同

1. 从整体功能上看，两者并没有大的差别。 都是将 Mapper（Spark 里是 ShuffleMapTask）的输出进行 Partition，不同的 Partition 送到不同的 Reducer（Spark 里 Reducer 可能是下一个 Stage 里的 ShuffleMapTask，也可能是 ResultTask）。Reducer 以内存作缓冲区，边 Shuffle 边 aggregate 数据，等到数据 aggregate 好以后进行 Reduce（Spark 里可能是后续的一系列操作）。
2. 从流程的上看，两者差别不小。 Hadoop MapReduce 是 sort-based，进入 combine 和 Reduce 的 records 必须先 sort。这样的好处在于 combine/Reduce可以处理大规模的数据。以前 Spark 默认选择的是 Hash-based，通常使用 HashMap 来对 Shuffle 来的数据进行合并，不会对数据进行提前排序。如果用户需要经过排序的数据，那么需要自己调用类似 sortByKey的操作。在 Spark 1.2 之后，sort-based 变为默认的 Shuffle 实现。
3. 从流程实现角度来看，两者也有不少差别。 Hadoop MapReduce 将处理流程划分出明显的几个阶段：Map, spill, merge, Shuffle, sort, Reduce 等。在 Spark 中，没有这样功能明确的阶段，只有不同的 Stage 和一系列的 transformation，所以 spill, merge, aggregate 等操作需要蕴含在 transformation 中。