#### 时间语义

* Event Time：事件创建的时间。一般就是数据本身携带的时间。

  优点：无论事件什么时候到达或者其怎么排序，最后处理 Event Time 将产生完全一致和确定的结果。

  缺点：处理 Event Time 时将会因为要等待一些无序事件而产生一些延迟。因为延迟时间有限，所以难以保证处理 Event Time 将产生完全一致和确定的结果。

* Ingestion Time：事件进入 Flink 数据流的时间 (source)

* Processing Time：事件被处理时机器的系统时间。

  优点：不需要数据流和机器之间的协调，提供了最好的性能和最低的延迟。

  缺点：在分布式和异步的环境下，Processing Time 不能提供确定性，因为它容易受到事件到达系统的速度（例如从消息队列）、事件在系统内操作流动的速度以及中断的影响。

设置时间策略：`env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);`

#### WaterMark

WaterMark 本质来说就是一个时间戳，代表着比这时间戳早的事件已经全部到达，不会再有比这时间戳还小的事件到达，这时候就会关闭窗口进行计算。

如果在程序里面收到了一个 Long.MAX_VALUE 这个数值的 WaterMark，就表示对应的那一条流的一个部分不会再有数据发过来了，它相当于就是一个终止的一个标志。

> 窗口的大小是不变的 (比如计算 11:00 - 12:00 的数据是确定的)，也就是说 window_end_time 是确定的。但什么时候触发要看 WaterMark。因为窗口不知道小于 window_end_time 的事件是否全部到了，WaterMark 这时候说比我时间戳小的都已经到了 (比如 12:05)。这时候比 WaterMark 小的 window_end_time 就可以关闭窗口进行计算了。

WaterMark 生成规则：

```java
// 当前事件时间的最大值 - 允许的延迟时间
new WaterMark(currentMaxTimestamp - maxOutOfOrderness)
```

窗口触发时机：`WaterMark  >= window_end_time`

数据元素在进入窗口时会分成两拨，一波是普通元素，进入 `processElement()` 方法处理，包括写入 WindowState、注册定时器等。

另一种是 WaterMark，会在 `processWaterMark()` 中被处理，首先判断有没有定时器小于 WaterMark，取出小于 WaterMark 的定时器并触发窗口计算；然后将所有数据流中最小的 WaterMark 广播到下游。

**在多并行度的情况下，WaterMark 会在数据流对齐后取所有数据流中最小的 WaterMark。**

<img src="/Users/licheng/Documents/Typora/Picture/image-20200709214431598.png" alt="image-20200709214431598" style="zoom: 67%;" />

Flink 中，数据处理中需要通过调用 DataStream 中的 assignTimestampsAndWaterMarks 方法来分配时间和水印，该方法可以传入两种参数，一个是 AssignerWithPeriodicWaterMarks，另一个是 AssignerWithPunctuatedWaterMarks。

* AssignerWithPunctuatedWaterMarks

   基于某些事件触发 WaterMark 的生成和发送基于事件向流里注入一个 WATERMARK，  每一个元素都有机会判断是否生成一个WATERMARK。如果得到的 WATERMARK  不为空并且比之前的大就注入流中实现AssignerWithPunctuatedWaterMarks 接口。

* AssignerWithPeriodicWaterMarks：周期性的 (一定时间间隔或者达到一定的记录条数) 产生一个 WaterMark。

  * BoundedOutOfOrdernessTimestampExtractor：该类用来发出滞后于数据时间的水印，可以传入一个时间代表着可以允许数据延迟到来的时间是多长。

  * AscendingTimestampExtractor：时间戳生成器，用于时间戳单调递增的数据流，

    > 因为是单调递增的数据，所以不需要 WaterMark 处理乱序数据。

  * IngestionTimeExtractor：依赖于机器系统时间，它在 extractTimestamp 和 getCurrentWaterMark 方法中是根据 `System.currentTimeMillis()` 来获取时间的，而不是根据事件的时间。如果这个时间分配器是在数据源进 Flink 后分配的，那么这个时间就和 Ingestion Time 一致了，所以命名也取的就是叫 IngestionTimeExtractor。

#### 延迟数据处理方式

* 丢弃：默认

* allowedLateness：表示允许数据延迟的时间 (在 WaterMark 允许延迟的基础上再次延迟的时间)。

  这个方法是在 WindowedStream 中的，用来设置允许窗口数据延迟的时间，超过这个时间的元素就会被丢弃，默认值是 0。

  > 当这个参数不为 0 时，窗口会在 trigger 触发时计算，但窗口不会被清除，窗口数据的计算结果会保存在 buffer 中，直到 windowEnd + allowedLateness <= WaterMark 时，会再次触发一次计算，相当于对之前计算结果的修正。

*  sideOutputLateData：收集迟到的数据。

  sideOutputLateData 这个方法同样是 WindowedStream 中的方法，该方法会将延迟的数据发送到给定 OutputTag 的 side output 中去，然后可以通过 `SingleOutputStreamOperator.getSideOutput(OutputTag)` 来获取这些延迟的数据。