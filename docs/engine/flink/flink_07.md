# 第七章 多流合并

Flink的多流合并的机制是以 {==FIFO==} 的方式合并多条流。

- union
  - 多条流的元素类型必须一样
  - 可以合并多条流：`stream1.union(stream2, stream3)`
- connect
  - 只能合并两条流
  - 两条流的元素的类型可以不一样
  - DataStream API
    - CoMapFunction<IN1, IN2, OUT>
      - map1
      - map2
    - CoFlatMapFunction<IN1, IN2, OUT>
      - flatMap1：来自第一条流的事件进入 CoFlatMapFunction，触发调用。
      - flatMap2：来自第二条流的事件进入 CoFlatMapFunction，触发调用。
  - 底层API
    - CoProcessFunction<IN1, IN2, OUT>
      - 两条流都必须进行 keyBy。
      - `processElement1`
      - `processElement2`
      - `onTimer`
      - 键控状态变量
    - 基于间隔的 join：`intervalJoin`
    - 基于窗口的 join