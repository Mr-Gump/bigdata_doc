# 第十二章 有限状态机

分两类：

- 确定性有限状态自动机（DFA）
- 非确定性有限状态自动机（NFA）

有限状态机的概念：

- 有限的状态（$S_1$，$S_2$）
  - $S_1$: 0为偶数个的状态
  - $S_2$: 0为奇数个的状态

- 事件（0，1）
- 有限状态机在接收到事件时，会发生状态的跳转。

![47](https://cos.gump.cloud/uPic/47.svg)

字符串：`1001`。

如果遍历完字符串，最终位于的状态是$S_1$，那么字符串中就有偶数个`0`。

[:link:flink cep官网链接](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/libs/cep/)