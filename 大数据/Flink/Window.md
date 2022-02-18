### 简介

Window 就是用来对一个无限的流设置一个有限的集合，在有界的数据集上进行操作的一种机制。Window 又可以分为基于时间(Time-based)的 Window 以及基于数量(Count-based)的 window。

### 分类

#### Time Window

Time Window 是根据时间对数据流划分窗口

* Tumbling Time Window

  将数据流按时间切分成不重叠的窗口，每一个事件只能属于一个窗口。如我们需要统计每一分钟中用户购买的商品的总数，需要将用户的行为事件按每一分钟进行切分。

  ```java
  lines.keyBy(0)
  .timeWindow(minutes(1))
  .sum(1);
  ```

* Sliding Time Window

  相比于滚动窗口，滑动窗口可以更加平滑的进行窗口聚合。比如，我们可以每30秒计算一次最近一分钟用户购买的商品总数。

  ```java
  lines.keyBy(0)
  .timeWindow(Time.minutes(1),Time.seconds(30))
  .sum(1);
  ```

#### Count Window

Count Window 是根据元素个数对数据流划分窗口的

* Tumbling Count Window

  将数据流按元素个数切分成不重叠的窗口，每一个事件只能属于一个窗口。如我们想要统计每100个用户的购买商品总数。

  ```java
  lines.keyBy(0).countWindow(100).sum(1);
  ```

* Sliding Count Window

  与 Sliding Time Window 含义类似，Sliding Count Window 也可以更加平滑的进行窗口聚合，只不过是基于元素数量。如每10个元素计算一次最近100个元素的总和。

  ```java
  lines.keyBy(0).countWindow(10,5).sum(1);
  ```

#### Session Window

将事件聚合到会话窗口中(一段用户持续活跃的周期)，由非活跃的间隙分隔开。通俗一点说，消息之间的间隔小于超时阈值(sessionGap)的，则被分配到同一个窗口，间隔大于阈值的，则被分配到不同的窗口。如需要计算每个用户在活跃期间总共购买的商品数量，如果用户30秒没有活动则视为会话断开。

```java
lines.keyBy(0)
.window(ProcessingTimeSessionWindows
.withGap(Time.seconds(30)));
```

### Window 组件

```
dataStream.keyBy(...)               <-  keyed versus non-keyed windows
.window(...)              <-  required: "assigner"
[.trigger(...)]            <-  optional: "trigger" (else default trigger)
[.evictor(...)]            <-  optional: "evictor" (else no evictor)
[.allowedLateness(...)]    <-  optional: "lateness" (else zero)
[.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
.reduce/aggregate/fold/apply()      <-  required: "function"
[.getSideOutput(...)]      <-  optional: "output tag"
```

* Window Assigner：用来决定某个元素被分配到哪个/哪些窗口中去。

* Trigger：触发器。确定一个窗口何时被计算或清除，每个窗口都会拥有一个自己的 Trigger。

* Evictor：可以译为“驱逐者”。在 Trigger 触发之后，在窗口被处理之前，Evictor 用来剔除窗口中不需要的元素，相当于一个 filter。

### Window 处理流程

<img src="C:\Users\admin\Typora\Picture\image-20200512185459320.png" alt="image-20200512185459320" style="zoom:67%;" />

首先上图中的组件都位于一个算子 (window operator) 中，数据流源源不断地进入算子，每一个到达的元素都会被交给 WindowAssigner。WindowAssigner 会决定元素被放到哪个或哪些窗口中。

每一个窗口都拥有一个属于自己的 Trigger，Trigger 上会有定时器，用来决定一个窗口何时能够被计算或清除。每当有元素加入到该窗口，或者之前注册的定时器超时了，那么 Trigger 都会被调用。

Trigger 返回结果：

* continue：不做任何操作。
* fire：处理窗口数据，窗口中的数据仍然保留不变，等待下次 Trigger fire 的时候再次执行计算。
* purge：移除窗口和窗口中的数据。
* fire + purge：处理之后移除窗口。

当 Trigger fire 了，窗口中的元素集合就会交给`Evictor`(如果指定了的话)。Evictor 主要用来遍历窗口中的元素列表，并决定最先进入窗口的多少个元素需要被移除。剩余的元素会交给用户指定的函数进行窗口的计算。如果没有 Evictor 的话，窗口中的所有元素会一起交给函数进行计算。

计算函数收到了窗口的元素，并计算出窗口的结果值，并发送给下游。窗口的结果值可以是一个也可以是多个。

DataStream API 上可以接收不同类型的计算函数，包括预定义的`sum()`,`min()`,`max()`，还有 `ReduceFunction`，`FoldFunction`，还有`WindowFunction`。`WindowFunction`是最通用的计算函数，其他的预定义的函数基本都是基于该函数实现的。

> Flink 对于一些聚合类的窗口计算 (如sum,min) 做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个 Result 值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改 result 值。这样可以大大降低内存的消耗并提升性能。但是如果用户定义了 Evictor，则不会启用对聚合窗口的优化，因为 Evictor 需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

#### 详细流程

* WindowOperator 接到消息以后，首先存到 WindowState，key 是 (key, window) 指定的二元组，value 是数据。

* 注册一个定时器，用于确定 Window 何时触发计算，将定时器放到优先队列中。

* 当 WindowOperator 收到 WaterMark 以后，取出集合中小于 WaterMark 的定时器，触发其 Window。触发的过程中将 WindowState 里面对应的数据取出来，发送给 WindowFunction 计算。

### 细粒度滑动窗口的性能问题

滑动窗口的窗口长度和步长的差值应该尽量小，WindowOperator 内部维护了窗口状态 WindowState。每一个元素会将其写入对应的 (key, window) 二元组所指定的状态中。如果在极端情况下，窗口长度为 24 小时，步长为 1 分钟，那么窗口个数就为 24 * 60  个，那么每个元素到来，更新 windowState 时都要遍历 1000 多个窗口并写入，开销是非常大的。

而且每一个 WindowState 都需要注册两个定时器：一是 Trigger 注册的定时器，用于决定窗口数据何时被触发计算；二是 `registerCleanupTimer()` 方法注册的清理定时器，用于在窗口彻底过期之后及时清理掉窗口的内部状态。这些定时器都是在堆内存的优先队列中，细粒度滑动窗口会造成维护的定时器增多，内存负担加重。