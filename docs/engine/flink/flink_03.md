# 第3章 DataStream API

## 3.1 自定义数据源

### 3.1.1 POJO CLASS

- 必须是公有类
- 所有字段必须是公有字段
- 必须有空构造器

### 3.1.2 SourceFunction

SourceFunction\<T> 的泛型是数据源中的数据的类型。

单并行度的数据源。并行度只能设置为 1。

- run 方法用来发送数据，由“程序启动”事件触发执行。
  - `ctx.collect` 向下游发送数据。
  - ctx.collectWithTimestamp 向下游发送数据，并指定数据的事件时间。
  - ctx.emitWatermark 向下游发送水位线事件。

- cancel方法在取消任务时执行，由“取消任务”事件触发执行。
  - 在 WebUI 上点击取消任务按钮
  - 在命令行 `flink cancel JobID`


> :memo:编写 Flink 程序时，要注意泛型和方法的参数类型！

其他自定义数据源 API

- ParallelSourceFunction\<T>：并行数据源
- RichSourceFunction\<T>：富函数版本
- RichParallelSourceFunction\<T>：并行数据源的富函数版本

## 3.2 基本转换算子

map, flatMap, filter

> :memo:代表算子：flatMap

基本转换算子都是{==无状态算子==}（在输入相同的情况下，输出一定相同）。

举个无状态函数的例子

> 数学函数：y = f(x)

```c
int add(int n) {
  return n + 1;
}

add(1); // => 2
add(1); // => 2
```

- MapFunction<IN, OUT> 的语义：针对流或者列表中的每一个元素，输出一个元素。
- FlatMapFunction<IN, OUT> 的语义：针对流或者列表中的每个元素，输出0个、1个或者多个元素。
- FilterFunction\<IN> 的语义：针对流或者列表中的每个元素，输出0个或者1个元素。

flatMap 是 map 和 filter 的泛化，也就是说可以使用 flatMap 来实现 map 和 filter 的功能。

## 3.3 逻辑分区算子

> :memo:代表算子：reduce

- keyBy 的作用：
  - 指定数据的 key。
  - 根据 key 计算出数据要去下游算子 reduce 的哪一个并行子任务。
  - 将数据发送到下游算子 reduce 的对应的并行子任务中。
- 终极目标：相同 key 的数据一定会发送到下游算子 reduce 的同一个并行子任务中。
- 不同 key 的数据也可能发送到下游算子 reduce 的同一个并行子任务中。

> keyBy 不是一个算子，因为不具备计算功能。keyBy 的作用只是为数据指定 key，并将数据路由到对应的 reduce 的并行子任务中。

reduce 是{==有状态算子==}（输入相同的情况下，输出不一定相同）。

举个有状态函数的例子。下面的函数将全局变量 `count` 作为内部状态维护。

```c
int count = 0;    // 初始化全局变量count
int add(int n) {
  count += n;
  return count;
}

add(1); // => 1
add(1); // => 2
add(1); // => 3
```

- ReduceFunction\<T> 的语义：第一条元素到来，不调用 `reduce` 方法，而是直接作为累加器保存，并将累加器输出。第二条以及之后的元素到来，调用 `reduce` 方法，和累加器进行累加操作并更新累加器，然后将累加器输出。reduce 方法定义的是输入元素和累加器的累加规则。

```java
输入数据的 key 对应的新累加器 reduce(输入数据的 key 对应的旧累加器，输入数据)
```

- 每个 key 都会维护自己的累加器，输入数据更新完累加器之后，直接被丢弃。
- reduce 只能在 keyBy 之后使用。

### 3.3.1 reduce 算子的并行子任务如何针对每个 key 维护各自的累加器呢？

<figure><img src="figure/15.svg" alt="reduce算子的并行子任务如何维护逻辑分区" style="width:100%;display:block;margin-left:auto;margin-right:auto;"><figcaption style="text-align:center;color:brown;">图-20 reduce算子的并行子任务如何维护逻辑分区</figcaption></figure>

假设我们有如下 Flink 代码：

```java
env
  .fromElements(1,2,3,4,5,6,7,8)
  .setParallelism(1)
  .keyBy(r -> r % 3)
  .reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer v1, Integer v2) throws Exception {
      return v1 + v2;
    }
  })
  .setParallelism(4)
  .print()
  .setParallelism(4);
```

那么 keyBy 对数据的路由方式是：

- 1,4,7 路由到 reduce 的第 3（索引为 2）个并行子任务
- 3,6 路由到 reduce 的第 3（索引为 2）个并行子任务
- 2,5,8 路由到 reduce 的第 4（索引为 3）个并行子任务

<figure><img src="figure/14.svg" alt="keyBy可能的一种路由方式" style="width:100%;display:block;margin-left:auto;margin-right:auto;"><figcaption style="text-align:center;color:brown;">图-21 keyBy的路由方式</figcaption></figure>

## 3.4 数据重分布

将数据发送到下游算子的不同的并行子任务。

- `shuffle()`：随机向下游的并行子任务发送数据。:memo:这里的 shuffle 和之前 keyBy 的 shuffle 不是一回事儿！
- `rebalance()`：将数据轮询发送到下游的{==所有==}并行子任务中。round-robin。
- `rescale()`：将数据轮询发送到下游的{==部分==}并行子任务中。用在下游算子的并行度是上游算子的并行度的整数倍的情况。round-robin。
- `broadcast()`：将数据广播到下游的所有并行子任务中。
- `global()`：将所有数据发送到下游的第一个（索引为 0）并行子任务中。
- `custom()`：自定义分区。可以自定义将某个 `key` 的数据发送到下游的哪一个并行子任务中去。

## 3.5 富函数

算子的每一个{==并行子任务==}都有自己的{==生命周期==}。

- `open` 方法：在算子的计算逻辑执行前执行一次，适合做一些初始化的工作（打开一个文件，打开一个网络连接，打开一个数据库的连接）。生命周期的开始。
- `close` 方法：在算子的计算逻辑执行完毕之后执行一次，适合做一些清理工作。（关闭一个文件，关闭网络连接，关闭数据库连接）。生命周期的结束。
- `getRuntimeContext()` 方法：用来获取算子的并行子任务的一些上下文信息。比如当前算子的并行子任务的索引等等。

举一些例子

- MapFunction :arrow_right: RichMapFunction
- FilterFunction :arrow_right: RichFilterFunction
- FlatMapFunction :arrow_right: RichFlatMapFunction
- ReduceFunction :arrow_right: RichReduceFunction
- SourceFunction :arrow_right: RichSourceFunction
- SinkFunction :arrow_right: RichSinkFunction

## 3.6 自定义输出

SinkFunction\<T>：泛型是要输出的数据的泛型

RichSinkFunction\<T>