### 写入数据流程

1. Client 访问 Zookeeper，获取元数据存储所在的 RegionServer。
2. 通过刚刚获取的地址访问对应的 RegionServer，拿到对应的表所在的 RegionServer 并访问。
3. Region Server 找到目标 Region，检查数据是否与 Schema 一致。
4. 先将更新写入 Hlog，然后将更新写入 Memstore，判断 Memstore 存储是否已满，如果存储已满则需要 flush 为 StoreFile 文件。

### 读取数据流程 (Region 寻址)

1. Client 请求 Zookeeper 获取 META 表所在的 RegionServer 的地址 get /hbase/meta-region-server

2. 客户端访问 META 表所在的 Region Server，从 META 表中查询到访问 RowKey 所在的 RegionServer。

3. 客户端从 RowKey 所在的 Region Server 上获取数据。

   > 如果已经读取过一次，则 root 表和.META都会缓存到本地，直接去对应的 RegionServer 读取数据。

### Region 切分

#### 切分策略

* ConstantSizeRegionSplitPolicy：当 Region 中最大的 store 的大小大于设置的阈值后就会切分。

  配置参数：`hbase.hregion.max.filesize`

  缺点：大小不好设置，比如 10 G，对大表友好，但对小表比较浪费。如果 1 G，那么会产生大量的 Region，不适合管理。

* SteppingSplitPolicy：如果 Region 个数等于 1，切分阈值为 flush size * 2，否则为 MaxRegionFileSize。

  配置参数：`hbase.hregion.max.filesize`

  >  MemStore 刷新到磁盘的大小默认为 128 M，所以 Region 切分的默认大小就为 256 M。

#### 切分流程

<img src="/Users/licheng/Documents/Typora/Picture/image-20200501184954820.png" alt="image-20200501184954820" style="zoom: 50%;" />

1. RegionServer 更改 Zookeeper 的 /hbase/region-in-transition 下该 Region 的状态为 SPLITTING。

3. RegionServer 在父 Region 的 HDFS 目录下创建一个名称为 .splits 的子目录。

4. RegionServer 关闭父 Region，强制将数据刷新到磁盘，并将这个 Region 标记为 offline 的状态。此时，落到这个 Region 的请求都会返回 NotServingRegionException 这个错误。

5. 在 .split 文件夹下新建两个子文件夹：daughter A、daughter B，并在文件夹中生成 reference 文件，分别指向父 Region 中对应文件。

   > Reference 文件内容第一部分为切分点信息，第二部分为一个 boolean 值，true 表示父文件的上半部分，false 表示父文件的下半部分。

6. RegionServer 将 daughter A、daughter B 拷贝到 HBase 根目录下，形成两个新的 Region。

7. RegionServer 发送一个 put 请求到 meta 表，请求将父 Region 中在 meta 表中的状态设置成 offline，并增加子 Region 的信息。

8. RegionServer 开启两个子 Region，并正式提供对外写服务。

9. RegionServer 修改 /hbase/region-in-transition/region-name 的 ZNode 的状态为 SPLIT。

> 切分过程不涉及到数据的移动。

### Region 合并

* 命令行：`merge_region 'ENCODED_REGIONNAME','ENCODED_REGIONNAME'`
* JAVA API：`admin.mergeRegions`

### HFile 合并

清除删除、过期、多余版本的数据；提高读写数据的效率。

#### 小合并 (MinorCompact)

将 Store 中多个 HFile 合并为一个 HFile。

配置参数：`hbase.hstore.compactionThreadhold`（最小 minor compaction 的文件个数，默认 3）。

合并步骤：

1. 分别读取出待合并的 StoreFile 文件的 KeyValues，并顺序地写入到位于 ./tmp 目录下的临时文件中。
2. 将临时文件移动到对应的 Region 目录中。
3. 将合并的输入文件路径和输出路径封装成 KeyValues 写入 WAL 日志，并打上 compaction 标记，最后强制自行 sync。
4. 将对应 Region 数据目录下的合并的输入文件全部删除，合并完成。

#### 大合并 (MajorCompact)

将一个 Store 中所有 HFile 合并为一个 Hfile，所有的数据删除操作都是在这个阶段完成的。

顺序重写全部的数据，重写数据的过程会略过做了删除标记的数据（之前删除的行、过期的版本、生存时间到期的数据都会被删除），拆分的父 Region 的数据也会迁移到拆分后的子 Region 上。

配置参数：`hbase.hregion.majorcompaction` (默认是 1 天，一般一周做一次)。