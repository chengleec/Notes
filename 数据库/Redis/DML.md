### 基本数据类型

#### String

最简单的类型，存储 Key/Value 数据。

##### 底层结构

SDS：简单动态字符串

##### 常用命令

`set key value`：添加/修改数据。

`get key`：获取数据。

`del key`：删除数据。

`incrby key increment`：让 key 增加指定值。

#### List

Lists 是有序列表，可以通过 List 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

##### 底层数据结构

* ZipList：元素长度小于 64 字节并且元素数量小于 512。
* LinkedList：元素长度大于 64 字节 或者元素数量大于 512。

##### 常用命令

`lpush/rpush key value1 value2 …`：添加/修改数据。

`lindex key 5`：获取某个下标的数据。

`lrange key 0 -1`：获取某个范围的数据。

#### Hash

##### 底层数据结构

* ZipList：元素长度小于 64 字节并且元素数量小于 512。
* Dict：元素长度大于 64 字节 或者元素数量大于 512。

##### 常用命令

`hset key k value`：添加/修改数据。

`hget key k`：获取数据。

`hdel key k`：删除数据。

`hexists key k`：判断某个 key 是否存在。

`hincrby key k increment`：让 key 增加指定值。

#### Set

Set 是无序集合，自动去重。可以用于求独立用户数等场景。

##### 底层数据结构

* IntSet：元素都是整形并且元素数量小于 512。
* Dict：元素不是整形或者元素数量大于 512。

##### 常用命令

`sadd key value`：添加数据。

`srem key value`：删除数据。

`scard key`：获取数据总量。

`smembers key`：获取全部数据。

`sismember key value`：判断是否包含某个元素。

#### Zset

ZSet 是排序的 Set，去重但可以排序，写进去的时候给一个分数，自动根据分数排序。

##### 底层数据结构

* ZipList：元素长度小于 64 字节并且元素数量小于 128。
* SkipList + Dict：元素长度大于 64 字节或者元素数量大于 128。

##### 常用命令

`zadd key score`：添加数据。

`zrange key 0 -1`：升序获取数据。

`zrevrange key 0 -1`：降序获取数据。

`zrem key value`：删除数据。

`zremrangebyscore key 5 10`：删除某个范围的数据。

### 查询命令

`keys *`：查询所有 key。

`keys it*`：查询所有以 it 开头的 key。

`keys *it`：查询所有以 it 结尾的 key。

`keys ?it`：查询第一个字符任意，后两个字符是 it 的 key。

`keys it?`：查询以 it 开头，最后一个字符任意的 key。

`keys u[st]er`：查询 user 或者 uter。

### 事务

* `multi`：开始事务。
* `exec`：执行事务。
* `discard`：取消事务。
* `watch key`：监视 key，如果事务执行前 key 发生变动，事务将被打断。
* `unwatch key`：取消监视 key。

#### 事务流程

使用  `multi` 命令开始事务，将操作放入队列缓存，然后通过 `exec` 命令执行事务。事务中任意命令执行失败，其余的命令依然被执行。

Redis 事务不满足原子性，在入队时出错，那么队列中所有命令都不会执行。如果入队成功，但执行时出错，那么其他命令会执行成功，只有这条命令会执行失败。所以 Redis 事务也不支持回滚。