# 第13章 Spark任务的执行

## 13.1 概述

### 13.1.1 任务切分和任务调度原理

![image-20230208193106448](https://cos.gump.cloud/uPic/image-20230208193106448.png)

![image-20230208193307884](https://cos.gump.cloud/uPic/image-20230208193307884.png)

### 13.1.2 本地化调度

任务分配原则：根据每个 Task 的优先位置，确定 Task 的 Locality（本地化）级别，本地化一共有五种，优先级由高到低顺序：

{==移动数据不如移动计算==}。

| 名称          | 解析                                                         |
| ------------- | ------------------------------------------------------------ |
| PROCESS_LOCAL | 进程本地化，task 和数据在同一个 Executor 中，性能最好。      |
| NODE_LOCAL    | 节点本地化，task 和数据在同一个节点中，但是 task 和数据不在同一个 Executor 中，数据需要在进程间进行传输。 |
| RACK_LOCAL    | 机架本地化，task 和数据在同一个机架的两个节点上，数据需要通过网络在节点之间进行传输。 |
| NO_PREF       | 对于 task 来说，从哪里获取都一样，没有好坏之分。             |
| ANY           | task 和数据可以在集群的任何地方，而且不在一个机架中，性能最差。 |

### 13.1.3 失败重试与黑名单机制

Task 运行失败会被告知给 TaskSetManager，对于失败的 Task，会记录它失败的次数，如果失败次数还没有超过最大重试次数，那么就把它放回待调度的 Task 池子中，否则整个 Application 失败。

失败同时会记录它上一次失败所在的 Executor Id 和 Host，使用黑名单机制，避免它被调度到上一次失败的节点上，起到一定的容错作用。黑名单记录 Task 上一次失败所在的 Executor Id 和 Host，以及其对应的“拉黑”时间，“拉黑”时间是指这段时间内不要再往这个节点上调度这个 Task 了。
