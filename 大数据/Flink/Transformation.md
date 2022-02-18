#### DataStream基本转换

![image-20200601095243948](/Users/licheng/Documents/Typora/Picture/image-20200601095243948.png)

#### 物理分组

![image-20200601095212871](/Users/licheng/Documents/Typora/Picture/image-20200601095212871.png)

#### Map

DataStream → DataStream，获取一个元素并生成一个元素

```java
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
```

#### FlatMap

DataStream → DataStream，获取一个元素并生成零个、一个或多个元素

```java
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
```

#### Filter

DataStream → DataStream，对每个元素都进行判断，返回为 true 的元素

```java
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
```

#### KeyBy

DataStream → KeyedStream，在逻辑上是基于 key 对流进行分区，相同的 Key 会被分到一个分区。底层使用 hash 函数对流进行分区。

```java
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
```

#### Reduce

KeyedStream → DataStream，返回单个结果值，reduce 每处理一个元素会创建一个新值。(聚合操作)

```java
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
```

#### Fold

KeyedStream → DataStream，给数据流一个初始值，将当前值与其组合后产生一个新值。

```java
DataStream<String> result = keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
});
```

#### Aggregate

KeyedStream → DataStream，用于实现自定义的聚合操作。该方法还可以添加一个 WindowFunction参数，将聚合后的结果带上其他信息输出。

```java
private static class AverageAggregate implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  // 创建一个初始累加器
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }
  @Override
  // 从输入中抽取某一项加到累加器上
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }
  @Override
  // 获取输出结果
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }
  @Override
  // 合并累加器结果（分区会有多个累加器）
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}
input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate());
```

Flink 实现了一部分聚合操作，例如 min、max、sum 等。 

```java
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
```

> max 和 maxBy 之间的区别在于 max 返回流中的最大值，但 maxBy 返回具有最大值的键， min 和 minBy 同理。

#### ProcessFunction

ProcessFunction 用于实现用户自定义的业务逻辑，通过 Context 可以访问 time、watermark 和 state 信息，使得编程更加灵活。

所有的 ProcessFunction 都继承自 RichFunction 接口，所以都有 open()、close() 和 getRuntimeContext() 等方法。

Flink提供了 8 个 Process Function：

* ProcessFunction：dataStream
* KeyedProcessFunction：KeyedStream
* CoProcessFunction：用于connect连接的流
* ProcessJoinFunction：用于join流操作
* BroadcastProcessFunction：用于广播
* KeyedBroadcastProcessFunction：keyBy之后的广播
* ProcessWindowFunction：窗口增量聚合
* ProcessAllWindowFunction：全窗口聚合

KeyedProcessFunction 额外提供了两个方法：

* `processElement(I value, Context ctx, Collector<O> out)`，流中的每一个元素都会调用这个方法，调用结果将会放在 Collector 中输出。Context 可以访问元素的时间戳，Key 以及 TImerService 时间服务。Context 还可以将结果输出到其他流(side outputs)。
* `onTimer(long timestamp, OnTimerContext ctx, Collector<O> out)`，这是一个回调函数。当之前注册的定时器触发时调用。参数 timestamp 为定时器所设定的触发的时间戳。Collector 为输出结果的集合。OnTimerContext 和 processElement 的 Context 参数一样，提供了一些上下文信息。

```java
// the source data stream
DataStream<Tuple2<String, String>> stream = ...;

// apply the process function onto a keyed stream
DataStream<Tuple2<String, Long>> result = stream
    .keyBy(0)
    .process(new CountWithTimeoutFunction());

/**
 * The data type stored in the state
 */
public class CountWithTimestamp {

    public String key;
    public long count;
    public long lastModified;
}

/**
 * The implementation of the ProcessFunction that maintains the count and timeouts
 */
public class CountWithTimeoutFunction extends KeyedProcessFunction<Tuple, Tuple2<String, String>, Tuple2<String, Long>> {

    /** The state that is maintained by this process function */
    private ValueState<CountWithTimestamp> state;

    @Override
    public void open(Configuration parameters) throws Exception {
        state = getRuntimeContext().getState(new ValueStateDescriptor<>("myState", CountWithTimestamp.class));
    }

    @Override
    public void processElement(
            Tuple2<String, String> value, 
            Context ctx, 
            Collector<Tuple2<String, Long>> out) throws Exception {

        // retrieve the current count
        CountWithTimestamp current = state.value();
        if (current == null) {
            current = new CountWithTimestamp();
            current.key = value.f0;
        }

        // update the state's count
        current.count++;

        // set the state's timestamp to the record's assigned event time timestamp
        current.lastModified = ctx.timestamp();

        // write the state back
        state.update(current);

        // schedule the next timer 60 seconds from the current event time
        ctx.timerService().registerEventTimeTimer(current.lastModified + 60000);
    }

    @Override
    public void onTimer(
            long timestamp, 
            OnTimerContext ctx, 
            Collector<Tuple2<String, Long>> out) throws Exception {

        // get the state for the key that scheduled the timer
        CountWithTimestamp result = state.value();

        // check if this is an outdated timer or the latest timer
        if (timestamp == result.lastModified + 60000) {
            // emit the state on timeout
            out.collect(new Tuple2<String, Long>(result.key, result.count));
        }
    }
}
```

#### Union

DataStream* → DataStream，Union 函数将两个或多个相同的数据流结合在一起。

```java
dataStream.union(otherStream1, otherStream2, ...);
```

#### Split

DataStream → SplitStream，Split 函数根据条件将流拆分为两个或多个流。

```java
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
```

#### Select

SplitStream → DataStream，从 SplitStream 中选择一个或者多个流。一般搭配 Split 使用。

```java
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
```

#### Connector

DataStream,DataStream → ConnectedStreams，通过连接**不同**或相同数据类型的数据流，然后创建一个新的连接数据流，如果连接的数据流也是一个 DataStream 的话，那么连接后的数据流为 ConnectedStreams。

```java
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...
ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
```

#### Start new chain

从当前算子开始一个新的 Task Chain。

```java
someStream.filter(...).map(...).startNewChain().map(...);
```

#### Disable chaining

禁止将当前算子分成两个 Task Chain。

```java
someStream.map(...).disableChaining();
```

#### Set slot sharing group

Slot 名称不同的 SubTask 不能运行在同一个 Slot 中，主要用于隔离 Slot。

```java
someStream.filter(...).slotSharingGroup("name");
```

