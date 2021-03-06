#### LSM Tree 

结构化合并树 (Log-Structured-Merge Tree)，它的核心思路其实非常简单，就是先将数据驻留在内存中，等到积累到一定阈值之后，再使用归并排序的方式将内存内的数据合并，然后追加到磁盘队尾。

最简单的 LSM 模型是 Two-Level LSM Tree。这个简易数据结构是由两个树状结构构成，这两颗树分别为 C0 和 C1。C0 比较小，并且全部驻于内存之中，而 C1 则驻于磁盘。一条新的记录先是从 C0 中插入，如果这一次的插入造成了 C0 数据量超出了阀值，那么 C0 中的某些数据片段则会迁出并合并到 C1 树中。其合并排序算法是批量的，由于是顺序存储，速度相当快。

然而通常情况下，每次持久化到硬盘中是一条独立的线程做的，并且生成单独的文件，因此 C1 树也不止一个文件，当文件数变多的时候，势必导致每一次查询都会涉及到大量文件的打开，每一次文件的打开都是对 I/O 的消耗。为了控制这种 **读放大** 的情况出现，LSM Tree 必须要考虑小文件合并的问题。

将 C1 树合并就是三层 LSM 树模型。

<img src="/Users/licheng/Documents/Typora/Picture/image-20200701161516469.png" alt="image-20200701161516469" style="zoom:50%;" />

##### B+ 树与 LSM-Tree 树区别

* LSM 树采用顺序写磁盘的方式，写入性能较高，B+ 树采用随机写磁盘的方式，需要先查找写入位置，效率较低。
* LSM 在磁盘中会生成很多小文件，每次查询都会涉及大量文件的打开，所以读性能比 B+ 树低。

#### HBase 对 LSM 树的优化

* BloomFilter：先判断 Hbase 中是否有这条数据，如果没有，直接返回。
* Compact：使用三层 LSM 树模型，定期合并 HFile，减少磁盘 IO，提高读性能。