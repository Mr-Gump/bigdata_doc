# 第四章 底层API

:memo:底层API（处理函数）都是{==富函数==}。

## 4.1 ProcessFunction

用来做基本转换。针对没有 `.keyBy` 的数据流，可以使用 `ProcessFunction` 接口，针对流中的每个元素输出 0 个、1 个或者多个元素。（加强版 `RichFlatMapFunction` ）

- `ProcessFunction<IN, OUT>`：IN 是输入的泛型，OUT 是输出的泛型
- `processElement`：每来一条数据，触发一次调用。
- 使用 `.process(new ProcessFunction<IN, OUT>)` 来调用。

## 4.2 KeyedProcessFunction

针对 keyBy 之后的键控流（ KeyedStream ），可以使用 KeyedProcessFunction

- `KeyedProcessFunction<KEY, IN, OUT>`：KEY 是 key 的泛型，IN 是输入的泛型，OUT 是输出的泛型。
- `processElement`：来一条数据，触发调用一次。
- `onTimer`：定时器。时间到达某一个时间戳触发调用。

![16](https://cos.gump.cloud/uPic/16.svg)

> :memo:定时器队列是定时器时间戳组成的数组。
>
> 数组经过了堆排序（优先队列）。

- 每个 key 都会维护自己的定时器，每个 key 都只能访问自己的定时器。就好像每个 key 都只能访问自己的累加器一样。
- 针对每个 key，在某个时间戳只能注册一个定时器，定时器不能重复注册，如果某个时间戳已经注册了定时器，那么再对这个时间戳注册定时器就不起作用了。{==幂等性==}。
- `.registerProcessingTimeTimer(ts)`：在机器时间戳 ts 注册了一个定时器（ onTimer ）。
- 维护的内部状态
  - 状态变量
  - 定时器
- 内部状态会每隔一段时间作为检查点保存到状态后端（例如 HDFS ）。
- processElement 方法和 onTimer 方法：这两个方法是{==原子性==}的，无法并发执行。某个时刻只能执行一个方法。因为这两个方法都有可能操作相同的状态变量。例如：到达了一个事件，此时 onTimer 正在执行，则必须等待 onTimer 执行完以后，再调用 processElement。再比如：到达了一个水位线，想触发 onTimer，但此时 processElement 正在执行，那么必须等待 processElement 执行完以后再执行 onTimer。
- {==当水位线到达 KeyedProcessFunction，如果这条水位线触发了 onTimer 的执行，则必须等待 onTimer 执行完以后，水位线才能向下游发送==}。
- {==当水位线到达 ProcessWindowFunction，如果这条水位线触发了 process 方法的执行，则必须等待 process 方法执行完以后，水位线才能向下游发送==}。

> :memo:在 KeyedProcessFunction 中，可以认为维护了多张 HashMap，每个状态变量的定义都会初始化一张 HashMap，同时还有一张维护每个 key 的定时器队列的 HashMap。

## 4.3 逻辑分区维护的状态-键控状态变量

每个 key 都会维护自己的状态变量。

- ValueState：值状态变量。类似 Java 的普通变量一样使用。
- ListState：列表状态变量。类似 Java 的 ArrayList 一样使用。
- MapState：字典状态变量。类似 Java 的 HashMap 一样使用。

### 4.3.1 ValueState-值状态变量

- 每个 key 都只能访问自己的状态变量，状态变量是每个 key 独有的。
- 在 `processElement(IN, CTX, OUT)` 方法中操作的状态变量，是输入数据 `IN` 的 key 所对应的状态变量。
- 状态变量是{==单例==}，只会被初始化一次。
- 当 Flink 程序启动时，会先去状态后端的检查点文件中寻找状态变量，如果找不到，则初始化。如果找到了，则直接读取。符合单例特性。
  !!! question "为什么 Flink 程序启动的时候，先去状态后端寻找状态变量呢？"
    因为 Flink 不知道程序是第一次启动，还是故障恢复后的启动。如果是故障恢复，则要去保存的检查点文件里寻找状态变量，恢复到最近一次检查点的状态。为了保证计算结果的正确性，状态变量必须被实现为单例模式。
- `getRuntimeContext().getState(状态描述符)` 方法通过状态描述符去状态后端的检查点文件中寻找状态变量，如果找不到，则初始化一个状态变量。
- 读取值状态变量中的值：`.value()` 方法
- 将值写入状态变量：`.update()` 方法
- 如何清空状态变量：`.clear()` 方法

![17](https://cos.gump.cloud/uPic/17.svg)

### 4.3.2 ListState-列表状态变量

> :material-warning:不要在 ValueState 中保存 ArrayList，应该使用 ListState

- `.get()` 方法：返回包含列表状态变量中所有元素的迭代器
- `.clear()` 方法：清空状态变量
- `.add()` 方法：添加元素
- `getRuntimeContext().getListState(列表状态描述符)`

![19](https://cos.gump.cloud/uPic/19.svg)

### 4.3.3 MapState-字典状态变量

> :material-warning:不要在 ValueState 中保存 HashMap，应该使用 MapState

- `.put(KEY, VALUE)` 方法：添加 KEY、VALUE 键值对
- `.get(KEY)` 方法：获取 KEY 的 VALUE
- `.contains(KEY)` 方法：检测 KEY 是否存在
- `.keys()`：返回所有 KEY 组成的集合

![18](https://cos.gump.cloud/uPic/18.svg)

## 4.4 窗口API

### 4.4.1 ProcessWindowFunction

使用在 `stream.keyBy().window()` 之后的流

- `ProcessWindowFunction<IN, OUT, KEY, WINDOW>`

  - `process` 方法：窗口闭合的时候触发调用

- Flink 中的窗口是左闭右开：[0,10)，当时间到达 9999ms 的时候，触发 process 函数的调用。

- 滚动时间窗口的计算公式，开了一个 5 秒钟的滚动窗口，7 秒钟到达的事件属于哪个窗口？

- 滚动窗口开始时间 = 时间戳 - 时间戳 % 窗口大小

  - windowStartTime = 1234ms - 1234 % 5000 = 0ms
  - windowStartTime = 7000ms - 7000 % 5000 = 5000ms

- 滑动窗口（窗口长度 10s，滑动距离 5s）

  - 首先计算第一个滑动窗口的开始时间（ 5s ）：

    - 窗口开始时间 = 时间戳 - 时间戳 % 滑动距离

    - 7s - 7s % 5s -> 5s

  - 然后计算其它滑动窗口的开始时间

    - ```
      for (start = 5s;
           start > 7s - windowSize(10s);
           start -= windowSlide(5s)) {
        ArrayList.add(start);
      }
      ```

    - ArrayList[0s, 5s] 中包含的就是事件所属的所有滑动窗口的开始时间

- 属于某个窗口的第一条数据到达以后才会开窗口

- 窗口内部状态：

  - 属于窗口的所有事件
  - 定时器：时间戳=窗口结束时间 - 1毫秒（因为是左闭右开区间），方法是 process 函数

> :material-warning:在只使用 ProcessWindowFunction 的情况下，process 方法的迭代器参数包含了属于窗口的所有数据，会对内存造成压力，那么应该怎么去优化呢？使用累加器的思想。

### 4.4.2 AggregateFunction

增量聚合函数，关键思想是在每个窗口中维护一个累加器。

- `AggregateFunction<IN, ACC, OUT>`
  - createAccumulator：创建空累加器，返回值的泛型是累加器的泛型
  - add：定义输入数据和累加器的聚合规则，返回值是聚合后的累加器
  - getResult：窗口闭合时发出聚合结果，返回值是将要发送的聚合结果
  - merge：一般不需要实现。只用在{==事件时间会话窗口==}的情况。

### 4.4.3 将AggregateFunction和ProcessWindowFunction结合使用

当窗口闭合时：AggregateFunction 将 getResult() 方法的返回值，发送给了 ProcessWindowFunction。

![25](https://cos.gump.cloud/uPic/25.svg)

ProcessWindowFunction 的 process 方法的迭代器参数中只有一个元素。

### 4.4.4 窗口的底层实现

AggregateFunction 和 ProcessWindowFunction 结合使用的{==底层实现==}

![26](https://cos.gump.cloud/uPic/26.svg)

窗口中的所有元素都保存在 List 中。

### 4.4.5 ProcessAllWindowFunction

```java
stream
  .windowAll(TumblingEventTimeWindow.of(...))
  .process(new ProcessAllWindowFunction(...))
```

直接对流进行开窗口，等价于将所有数据keyBy到同一条流，然后进行开窗口。

```java
stream
  .keyBy(r -> 1)
  .window()
  .process(new ProcessWindowFunction())
```

### 4.4.6 触发器

> 窗口大小是 1 天，但想每隔 10 秒钟计算一次窗口中的统计指标，怎么办？

触发器可以在窗口关闭之前，触发窗口的计算操作。

```java
stream
  .keyBy()
  .window()
  [.allowedLateness()] // 可选
  [.sideOutput()]      // 可选
  [.trigger()]         // 可选
  .process()/aggregate()
```

触发器触发的是 trigger 后面的 process / aggregate 方法。

Trigger<T, W>：T 是窗口中的元素类型，W 是窗口类型。

TriggerResult

- CONTINUE：什么都不做
- FIRE：触发窗口的计算
- PURGE：清空窗口中的元素
- FIRE_AND_PURGE：触发窗口计算并清空窗口中的元素

Trigger 中的核心方法

- onElement：每来一条数据调用一次
- onProcessingTime：处理时间定时器
- onEventTime：事件时间定时器
- clear：窗口闭合时触发调用
