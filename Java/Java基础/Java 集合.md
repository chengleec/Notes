### Java 集合架构

<img src="/Users/licheng/Documents/Typora/Picture/image-20200823220207946.png" alt="image-20200823220207946" style="zoom:67%;" />

### Collection

#### ArrayList

底层基于数组实现，支持对元素的随机访问，但添加和删除数据需要移动整个数组，时间复杂度为 $O(n)$。

默认初始大小为 10，每次扩容为旧容量的 1.5 倍，需要将旧 ArrayList 中的数据复制到新 ArrayList。

#### LinkedList

底层基于双向链表实现，添加和删除数据的时间复杂度为 $O(1)$。

#### HashSet

无序不重复。底层使用 HashMap 实现，方法实现直接调用 HashMap 方法。

#### LinkedHashSet

有序不重复。底层使用 LinkedHashMap 实现，多了一个链表用来维护元素插入顺序。

#### TreeSet

底层使用 TreeMap 实现，不重复，无索引，默认按照元素大小升序排序。

****

### Map

#### HashMap

底层基于数组+链表/红黑树实现，当链表长度大于 8 时会转换成红黑树。

**HashMap 保存自定义对象的时候需要重写 hashCode 和 equals 方法**，如果不重写的话调用的是 Object 的 hashCode 和 equals 方法，那么他比较的是对象的地址，而不是对象的内容。

##### put 方法

* 判断数组是否为空，为空进行初始化。
* 如果不为空，计算 key 的 hash 值，通过 `(n - 1) & hash` 计算数组下标 index。
* 如果 table[index] 为空就构造一个 Node 节点存放在 table[index] 中。
* 如果存在数据，判断 key 是否相等，如果相等就用新数据覆盖原数据。
* 如果不相等，判断当前节点类型是不是树型节点，如果是树型节点，构造一个树型节点插入红黑树中。
* 如果不是红黑树，创建普通 Node 节点加入链表中。
* 判断链表长度是否大于 8，如果大于，首先检查数组长度是否小于 64，如果小于 64 则扩容。如果等于 64 就将链表转换为红黑树。

##### get 方法

* 计算 key 的 hash 值，然后通过 `(n - 1) & hash` 找到元素存放的位置。
* 如果与数组中元素相等，直接返回其 value。
* 如果不相等，判断其元素类型。如果是红黑树，就在树中查找；如果是链表，就在链表中查找。

##### JDK 1.7 与 JDK 1.8 的区别

* 1.7 底层基于数组加链表实现，1.8 底层基于数组 + 链表/红黑树实现。
* 1.7 Rehash 时采用的是头插法，可能会出现死循环，1.8 Rehash 采用尾插法。

#### LinkedHashMap

与 HashMap 多了一个链表，用来维护元素插入顺序。

#### TreeMap

底层基于红黑树实现，默认按照元素大小升序排序。

****

### Java.util.Concurrent

#### CopyOnWriteArrayList

CopyOnWriteArrayList 是一个线程安全的 ArrayList。适用于读操作多的场景。

CopyOnWriteArrayList 在写操作的时候，会先加锁，然后将数组复制一份，对复制的数组进行操作，操作完毕后再将 list 地址引用指向修改后的新数组地址。对所有的读操作都不加锁。

#### ConcurrentHashMap

底层基于数组 + 链表 / 红黑树实现，并发安全通过 CAS + Synchronized 保证。

##### put 方法

* 判断数组是否为空，为空进行初始化。
* 如果不为空，计算 key 的 hash 值，通过 `(n - 1) & hash` 计算数组下标 index。
* 查看 table[index] 是否存在数据，没有数据就使用 CAS 尝试写入，失败则自旋保证成功。
* 如果当前位置的 `hashcode == MOVED == -1`，则当前位置正在扩容，需要帮助其扩容。
* 如果都不满足，则利用 Synchronized 锁写入数据。
* 如果链表结点数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。

##### get 方法

* 计算 key 的 hash 值，然后通过 `(n - 1) & hash` 找到元素存放的位置。
* 如果与数组中元素相等，直接返回其 value。
* 如果不相等，判断其元素类型。如果是红黑树，就在树中查找；如果是链表，就在链表中查找。

##### JDK 1.7 与 JDK 1.8 的区别

 JDK 1.7 采用的是 Segment 数组 + HashEntry 数组 + 链表的形式实现的，写数据时只锁住 Segment 数组的一部分，提高并发。

#### Queue

##### ArrayBlockingQueue

基于数组实现的有界阻塞队列。ArrayBlockingQueue 一旦创建，容量不能改变。

采用 FIFO 调度，并发控制采用 Reentrantlock + Condition 来控制，支持公平锁和非公平锁。

##### LinkedBlockingQueue

基于单向链表实现的阻塞队列，默认长度为 `Integer.MAX_VALUE`。

可以作为有界队列，也可以作为无界队列，采用 FIFO 调度。

##### PriorityBlockingQueue

支持线程优先级的无界阻塞队列。默认情况下元素采用自然顺序进行排序，也可以自定义 `compareTo()` 方法指定元素排列规则。

底层由 ReentrantLock + Condition + 二叉堆实现。

##### DelayQueue

封装了 PriorityBlockingQueue、支持元素延迟获取的无界队列，在创建元素时，可以指定多久才能从队列中获取当前元素，只有延迟期满后才能从队列中获取。

PriorityBlockingQueue 会对元素排序，延迟时间短的放在前面，如果延迟时间相同，就按 FIFO 排序。

##### SynchronousQueue

一个不存储元素的阻塞队列，每一个 put 操作必须等待 take 操作，否则不能添加元素。支持公平锁和非公平锁。

#### AQS

