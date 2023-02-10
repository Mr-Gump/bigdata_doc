# 水位线（Watermark）

## 事件时间和窗口

在实际应用中，一般会采用事件时间语义。而水位线，就是基于事件时间提出的概念。所 以在介绍水位线之前，我们首先来梳理一下事件时间和窗口的关系。

一个数据产生的时刻，就是流处理中事件触发的时间点，这就是“事件时间”，一般都会以时间戳的形式作为一个字段记录在数据里。这个时间就像商品的“生产日期”一样，一旦产生就是固定的，印在包装袋上，不会因为运输辗转而变化。如果我们想要统计一段时间内的数 据，需要划分时间窗口，这时只要判断一下时间戳就可以知道数据属于哪个窗口了。

明确了一个数据的所属窗口，还不能直接进行计算。因为窗口处理的是有界数据，我们需要等窗口的数据都到齐了，才能计算出最终的统计结果。那什么时候数据就都到齐了呢？对于时间窗口来说这很明显：到了窗口的结束时间，自然就应该收集到了所有数据，就可以触发计算输出结果了。比如我们想统计 8 点~9 点的用户点击量，那就是从 8 点开始收集数据，到 9 点截止，将收集的数据做处理计算。这有点类似于班车，如图所示，每小时发一班，那么 8 点之后来的人都会上同一班车，到 9 点钟准时发车；9 点之后来的人，就只好等下一班 10 点发的车了。

<figure markdown>
  ![image-20230210194719721](https://cos.gump.cloud/uPic/image-20230210194719721.png)
  <figcaption>车站待发的班车</figcaption>
</figure>
当然，我们现在处理的数据本身是有时间戳的。所以为了更清楚地解释，我们将“赶班车”这个例子中的人，换成带着生产日期的商品。所以现在我们班车的主要任务是运送货物，一辆车就只装载 1 小时内生产出的所有商品，货到齐了就发车。比如某辆车要装的是 8 点到 9 点的所有商品，那货什么时候到齐呢？自然可以想到，到 9 点钟的时候商品就到齐了，可以发车了。

!!! question "关键问题"

    “9 点钟发车”，到底是看谁的表来定时间？

在处理时间语义下，都是以当前任务所在节点的系统时间为准的。这就相当于每辆车里都 挂了一个钟，司机看到到了 9 点就直接发车。这种方式简单粗暴容易实现，但因为车上的钟是 独立运行的，以它为标准就不能准确地判断商品的生产时间。在分布式环境下，这样会因为网 络传输延迟的不确定而导致误差。比如有些商品在 8 点 59 分 59 秒生产出来，可是从下生产线 到运至车上又要花费几秒，那就赶不上 9 点钟这班车了。而且现在分布式系统中有很多辆 9 点发的班车，所以同时生产出的一批商品，需要平均分配到不同班车上，可这些班车距离有近 有远、上面挂的钟有快有慢，这就可能导致有些商品上车了、有些却被漏掉；先后生产出的商 品，到达车上的顺序也可能乱掉：统计结果的正确性受到了影响。

所以在实际中我们往往需要以事件时间为准。如果考虑事件时间，情况就复杂起来了。现 在不能直接用每辆车上挂的钟（系统时间），又没有统一的时钟，那该怎么确定发车时间呢？

现在能利用的，就只有商品的生产时间（数据的时间戳）了。我们可以这样思考：一般情 况下，商品生产出来之后，就会立即传送到车上；所以商品到达车上的时间（系统时间）应该稍稍滞后于商品的生产时间（数据时间戳）。如果不考虑传输过程的一点点延迟，我们就可以 直接用商品生产时间来表示当前车上的时间了。如图所示，到达车上的商品，生产时间是 8 点 05 分，那么当前车上的时间就是 8 点 05 分；又来了一个 8 点 10 分生产的商品，现在车 上的时间就是 8 点 10 分。我们直接用数据的时间戳来指示当前的时间进展，窗口的关闭自然 也是以数据的时间戳等于窗口结束时间为准，这就相当于可以不受网络传输延迟的影响了。像 之前所说 8 点 59 分 59 秒生产出来的商品，到车上的时候不管实际时间（系统时间）是几点， 我们就认为当前是 8 点 59 分 59 秒，所以它总是能赶上车的；而 9 点这班车，要等到 9 点整生 产的商品到来，才认为时间到了 9 点，这时才正式发车。这样就可以得到正确的统计结果了。

<figure markdown>
  ![image-20230210194926232](https://cos.gump.cloud/uPic/image-20230210194926232.png)
  <figcaption>事件时间语义下窗口的开启和关闭（窗口时间区间为左闭右开，即包含 8 点不包含 9 点）</figcaption>
</figure>

在这个处理过程中，我们其实是基于数据的时间戳，自定义了一个“逻辑时钟”。这个时 钟的时间不会自动流逝；它的时间进展，就是靠着新到数据的时间戳来推动的。这样的好处在 于，计算的过程可以完全不依赖处理时间（系统时间），不论什么时候进行统计处理，得到的 结果都是正确的。比如双十一的时候系统处理压力大，我们可能会把大量数据缓存在 Kafka 中；过了高峰时段之后再读取出来，在几秒之内就可以处理完几个小时甚至几天的数据，而且 依然可以按照数据产生的时间段进行统计，所有窗口都能收集到正确的数据。而一般实时流处 理的场景中，事件时间可以基本与处理时间保持同步，只是略微有一点延迟，同时保证了窗口 计算的正确性。

### 什么是水位线

在事件时间语义下，我们不依赖系统时间，而是基于数据自带的时间戳去定义了一个时钟， 用来表示当前时间的进展。于是每个并行子任务都会有一个自己的逻辑时钟，它的前进是靠数 据的时间戳来驱动的。

但在分布式系统中，这种驱动方式又会有一些问题。因为数据本身在处理转换的过程中会 变化，如果遇到窗口聚合这样的操作，其实是要攒一批数据才会输出一个结果，那么下游的数 据就会变少，时间进度的控制就不够精细了。另外，数据向下游任务传递时，一般只能传输给 一个子任务（除广播外），这样其他的并行子任务的时钟就无法推进了。例如一个时间戳为 9 点整的数据到来，当前任务的时钟就已经是 9 点了；处理完当前数据要发送到下游，如果下游 任务是一个窗口计算，并行度为 3，那么接收到这个数据的子任务，时钟也会进展到 9 点，9 点结束的窗口就可以关闭进行计算了；而另外两个并行子任务则时间没有变化，不能进行窗口 计算。

所以我们应该把时钟也以数据的形式传递出去，告诉下游任务当前时间的进展；而且这个 时钟的传递不会因为窗口聚合之类的运算而停滞。一种简单的想法是，在数据流中加入一个时 钟标记，记录当前的事件时间；这个标记可以直接广播到下游，当下游任务收到这个标记，就 可以更新自己的时钟了。由于类似于水流中用来做标志的记号，在 Flink 中，这种用来衡量事 件时间（Event Time）进展的标记，就被称作“水位线”（Watermark）。

具体实现上，水位线可以看作一条特殊的数据记录，它是插入到数据流中的一个标记点， 主要内容就是一个时间戳，用来指示当前的事件时间。而它插入流中的位置，就应该是在某个 数据到来之后；这样就可以从这个数据中提取时间戳，作为当前水位线的时间戳了。


<figure markdown>
  ![image-20230210195027701](https://cos.gump.cloud/uPic/image-20230210195027701.png)
  <figcaption>每条数据后插入一个水位线</figcaption>
</figure>

如图所示，每个事件产生的数据，都包含了一个时间戳，我们直接用一个整数表示。 这里没有指定单位，可以理解为秒或者毫秒（方便起见，下面讲述统一认为是秒）。当产生于 2 秒的数据到来之后，当前的事件时间就是 2 秒；在后面插入一个时间戳也为 2 秒的水位线， 随着数据一起向下游流动。而当 5 秒产生的数据到来之后，同样在后面插入一个水位线，时间 戳也为 5，当前的时钟就推进到了 5 秒。这样，如果出现下游有多个并行子任务的情形，我们 只要将水位线广播出去，就可以通知到所有下游任务当前的时间进度了。

水位线就像它的名字所表达的，是数据流中的一部分，随着数据一起流动，在不同任务之 间传输。这看起来非常简单；接下来我们就进一步探讨一些复杂的状况。

#### 1. 有序流中的水位线

在理想状态下，数据应该按照它们生成的先后顺序、排好队进入流中；也就是说，它们处 理的过程会保持原先的顺序不变，遵守先来后到的原则。这样的话我们从每个数据中提取时间 戳，就可以保证总是从小到大增长的，从而插入的水位线也会不断增长、事件时钟不断向前推 进。

实际应用中，如果当前数据量非常大，可能会有很多数据的时间戳是相同的，这时每来一 条数据就提取时间戳、插入水位线就做了大量的无用功。而且即使时间戳不同，同时涌来的数 据时间差会非常小（比如几毫秒），往往对处理计算也没什么影响。所以为了提高效率，一般 会每隔一段时间生成一个水位线，这个水位线的时间戳，就是当前最新数据的时间戳，如图所示。所以这时的水位线，其实就是有序流中的一个周期性出现的时间标记。

<figure markdown>
  ![image-20230210195103858](https://cos.gump.cloud/uPic/image-20230210195103858.png)
  <figcaption>有序流中周期性插入水位线</figcaption>
</figure>

这里需要注意的是，水位线插入的“周期”，本身也是一个时间概念。在当前事件时间语 义下，假如我们设定了每隔 100ms 生成一次水位线，那就是要等事件时钟推进 100ms 才能插 入；但是事件时钟本身的进展，本身就是靠水位线来表示的——现在要插入一个水位线，可前 提又是水位线要向前推进 100ms，这就陷入了死循环。所以对于水位线的周期性生成，周期时 间是指处理时间（系统时间），而不是事件时间。

#### 2. 乱序流中的水位线

有序流的处理非常简单，看起来水位线也并没有起到太大的作用。但这种情况只存在于理 想状态下。我们知道在分布式系统中，数据在节点间传输，会因为网络传输延迟的不确定性， 导致顺序发生改变，这就是所谓的“乱序数据”。

这里所说的“乱序”（out-of-order），是指数据的先后顺序不一致，主要就是基于数据的产 生时间而言的。如图所示，一个 7 秒时产生的数据，生成时间自然要比 9 秒的数据早；但 是经过数据缓存和传输之后，处理任务可能先收到了 9 秒的数据，之后 7 秒的数据才姗姗来迟。

!!! question 

    这时如果我们希望插入水位线，来指示当前的事件时间进展，又该怎么做呢？


<figure markdown>
  ![image-20230210195150874](https://cos.gump.cloud/uPic/image-20230210195150874.png)
  <figcaption>乱序流</figcaption>
</figure>

最直观的想法自然是跟之前一样，我们还是靠数据来驱动，每来一个数据就提取它的时间 戳、插入一个水位线。不过现在的情况是数据乱序，所以有可能新的时间戳比之前的还小，如 果直接将这个时间的水位线再插入，我们的“时钟”就回退了——水位线就代表了时钟，时光 不能倒流，所以水位线的时间戳也不能减小。

解决思路也很简单：我们插入新的水位线时，要先判断一下时间戳是否比之前的大，否则 就不再生成新的水位线，如图所示。也就是说，只有数据的时间戳比当前时钟大，才能推 动时钟前进，这时才插入水位线。

<figure markdown>
  ![image-20230210195204613](https://cos.gump.cloud/uPic/image-20230210195204613.png)
  <figcaption>乱序流中的水位线</figcaption>
</figure>

如果考虑到大量数据同时到来的处理效率，我们同样可以周期性地生成水位线。这时只需 要保存一下之前所有数据中的最大时间戳，需要插入水位线时，就直接以它作为时间戳生成新 的水位线，如图所示。

<figure markdown>
  ![image-20230210195221569](https://cos.gump.cloud/uPic/image-20230210195221569.png)
  <figcaption>乱序流中周期性生成水位线</figcaption>
</figure>

这样做尽管可以定义出一个事件时钟，却也会带来一个非常大的问题：我们无法正确处理 “迟到”的数据。在上面的例子中，当 9 秒产生的数据到来之后，我们就直接将时钟推进到了 9 秒；如果有一个窗口结束时间就是 9 秒（比如，要统计 0~9 秒的所有数据），那么这时窗口 就应该关闭、将收集到的所有数据计算输出结果了。但事实上，由于数据是乱序的，还可能有 时间戳为 7 秒、8 秒的数据在 9 秒的数据之后才到来，这就是“迟到数据”（late data）。它们 本来也应该属于 0~9 秒这个窗口，但此时窗口已经关闭，于是这些数据就被遗漏了，这会导 致统计结果不正确。

如果用之前我们类比班车的例子，现在的状况就是商品不是按照生产时间顺序到来的，所 以有可能出现这种情况：9 点生产的商品已经到了，我们认为已经到了 9 点，所以直接发车； 但是可能还会有 8 点 59 分 59 秒生产的商品迟到了，没有赶上这班车。那怎么解决这个问题呢？

其实我们有很多生活中的经验。假如是一个团队出去团建，那肯定希望每个人都不能落下； 如果有人因为堵车没能准时到车上，我们可以稍微等一会儿。9 点发车，我们可以等到 9 点 10 分，等人都到齐了再出发。当然，实际应用的网络环境不可能跟北京的交通一样堵，所以不需 要等那么久，或许只要等一两秒钟就可以了。具体在商品班车的例子里，我们可以多等 2 秒钟， 也就是当生产时间为 9 点零 2 秒的商品到达，时钟推进到 9 点零 2 秒，这时就认为所有 8 点到 9 点生产的商品都到齐了，可以正式发车。不过这样相当于更改了发车时间，属于“违规操作”。 为了做到形式上仍然是 9 点发车，我们可以更改一下时钟推进的逻辑：当一个商品到达时，不 要直接用它的生产时间作为当前时间，而是减上两秒，这就相当于把车上的逻辑时钟调慢了。 这样一来，当 9 点生产的商品到达时，我们当前车上的时间是 8 点 59 分 58 秒，还没到发车时 间；当 9 点零 2 秒生产的商品到达时，车上时间刚好是 9 点，这时该到的商品都到齐了，准时 发车就没问题了。

回到上面的例子，为了让窗口能够正确收集到迟到的数据，我们也可以等上 2 秒；也就是 用当前已有数据的最大时间戳减去 2 秒，就是要插入的水位线的时间戳，如图所示。这 样的话，9 秒的数据到来之后，事件时钟不会直接推进到 9 秒，而是进展到了 7 秒；必须等到 11 秒的数据到来之后，事件时钟才会进展到 9 秒，这时迟到数据也都已收集齐，0~9 秒的窗 口就可以正确计算结果了。

<figure markdown>
  ![image-20230210195245427](https://cos.gump.cloud/uPic/image-20230210195245427.png)
  <figcaption>乱序流中“等 2 秒”策略</figcaption>
</figure>

如果仔细观察就会看到，这种“等 2 秒”的策略其实并不能处理所有的乱序数据。比如 22 秒的数据到来之后，插入的水位线时间戳为 20，也就是当前时钟已经推进到了 20 秒；对于 10~20 秒的窗口，这时就该关闭了。但是之后又会有 17 秒的迟到数据到来，它本来应该属于 10~20 秒窗口，现在却被遗漏丢弃了。那又该怎么办呢？

既然现在等 2 秒还是等不到 17 秒产生的迟到数据，那自然我们可以试着多等几秒，也就 是把时钟调得更慢一些。最终的目的，就是要让窗口能够把所有迟到数据都收进来，得到正确 的计算结果。对应到水位线上，其实就是要保证，当前时间已经进展到了这个时间戳，在这之 后不可能再有迟到数据来了。

下面是一个示例，我们可以使用周期性的方式生成正确的水位线。

<figure markdown>
  ![image-20230210195303773](https://cos.gump.cloud/uPic/image-20230210195303773.png)
  <figcaption>乱序流中周期性生成正确的水位线</figcaption>
</figure>

如图所示，第一个水位线时间戳为 7，它表示当前事件时间是 7 秒，7 秒之前的数据 都已经到齐，之后再也不会有了；同样，第二个、第三个水位线时间戳分别为 12 和 20，表示 11 秒、20 秒之前的数据都已经到齐，如果有对应的窗口就可以直接关闭了，统计的结果一定 是正确的。这里由于水位线是周期性生成的，所以插入的位置不一定是在时间戳最大的数据后 面。

另外需要注意的是，这里一个窗口所收集的数据，并不是之前所有已经到达的数据。因为数据属于哪个窗口，是由数据本身的时间戳决定的，一个窗口只会收集真正属于它的那些数据。 也就是说，上图中尽管水位线 W(20)之前有时间戳为 22 的数据到来，10~20 秒的窗口中也不会收集这个数据，进行计算依然可以得到正确的结果。关于窗口的原理，我们会在后面继续展开讲解。

#### 3. 水位线的特性

现在我们可以知道，水位线就代表了当前的事件时间时钟，而且可以在数据的时间戳基础 上加一些延迟来保证不丢数据，这一点对于乱序流的正确处理非常重要。 我们可以总结一下水位线的特性：

+ 水位线是插入到数据流中的一个标记，可以认为是一个特殊的数据
+ 水位线主要的内容是一个时间戳，用来表示当前事件时间的进展
+ 水位线是基于数据的时间戳生成的
+ 水位线的时间戳必须单调递增，以确保任务的事件时间时钟一直向前推进
+ 水位线可以通过设置延迟，来保证正确处理乱序数据
+ 一个水位线 Watermark(t)，表示在当前流中事件时间已经达到了时间戳 t, 这代表 t 之 前的所有数据都到齐了，之后流中不会出现时间戳 t’ ≤ t 的数据

水位线是 Flink 流处理中保证结果正确性的核心机制，它往往会跟窗口一起配合，完成对 乱序数据的正确处理。关于这部分内容，我们会稍后进一步展开讲解。

### 6.2.3 如何生成水位线

上一节中我们讲到，水位线是用来保证窗口处理结果的正确性的，如果不能正确处理所有 乱序数据，可以尝试调大延迟的时间。那在实际应用中，到底应该怎样生成水位线呢？本节我 们就来讨论这个问题。

#### 1. 生成水位线的总体原则

我们知道，完美的水位线是“绝对正确”的，也就是一个水位线一旦出现，就表示这个时 间之前的数据已经全部到齐、之后再也不会出现了。而完美的东西总是可望不可即，我们只能 尽量去保证水位线的正确。如果对结果正确性要求很高、想要让窗口收集到所有数据，我们该 怎么做呢？

一个字，等。由于网络传输的延迟不确定，为了获取所有迟到数据，我们只能等待更长的 时间。作为筹划全局的程序员，我们当然不会傻傻地一直等下去。那到底等多久呢？这就需要 对相关领域有一定的了解了。比如，如果我们知道当前业务中事件的迟到时间不会超过 5 秒， 那就可以将水位线的时间戳设为当前已有数据的最大时间戳减去 5 秒，相当于设置了 5 秒的延 迟等待。

更多的情况下，我们或许没那么大把握。毕竟未来是没有人能说得准的，我们怎么能确信 未来不会出现一个超级迟到数据呢？所以另一种做法是，可以单独创建一个 Flink 作业来监控 事件流，建立概率分布或者机器学习模型，学习事件的迟到规律。得到分布规律之后，就可以 选择置信区间来确定延迟，作为水位线的生成策略了。例如，如果得到数据的迟到时间服从 μ=1，σ=1 的正态分布，那么设置水位线延迟为 3 秒，就可以保证至少 97.7%的数据可以正确 处理。

如果我们希望计算结果能更加准确，那可以将水位线的延迟设置得更高一些，等待的时间 越长，自然也就越不容易漏掉数据。不过这样做的代价是处理的实时性降低了，我们可能为极 少数的迟到数据增加了很多不必要的延迟。

如果我们希望处理得更快、实时性更强，那么可以将水位线延迟设得低一些。这种情况下， 可能很多迟到数据会在水位线之后才到达，就会导致窗口遗漏数据，计算结果不准确。对于这 些 “漏网之鱼”，Flink 另外提供了窗口处理迟到数据的方法，我们会在后面介绍。当然，如 果我们对准确性完全不考虑、一味地追求处理速度，可以直接使用处理时间语义，这在理论上 可以得到最低的延迟。

所以 Flink 中的水位线，其实是流处理中对低延迟和结果正确性的一个权衡机制，而且把 控制的权力交给了程序员，我们可以在代码中定义水位线的生成策略。接下来我们就具体了解 一下水位线在代码中的使用。

#### 2. 水位线生成策略（Watermark Strategies）

在 Flink 的 DataStream API 中 ， 有 一 个 单 独 用 于 生 成 水 位 线 的 方 法：.assignTimestampsAndWatermarks()，它主要用来为流中的数据分配时间戳，并生成水位线 来指示事件时间：

```java
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(WatermarkStrategy<T> watermarkStrategy)
```

具体使用时，直接用 DataStream 调用该方法即可，与普通的 transform 方法完全一样。

```java
DataStream<Event> stream = env.addSource(new ClickSource());
DataStream<Event> withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(<watermark strategy>);
```

这里读者可能有疑惑：不是说数据里已经有时间戳了吗，为什么这里还要“分配”呢？这是 因为原始的时间戳只是写入日志数据的一个字段，如果不提取出来并明确把它分配给数据， Flink 是无法知道数据真正产生的时间的。当然，有些时候数据源本身就提供了时间戳信息， 比如读取 Kafka 时，我们就可以从 Kafka 数据中直接获取时间戳，而不需要单独提取字段分配了。

.assignTimestampsAndWatermarks() 方法需要传入一个 WatermarkStrategy 作为参数，这就 是 所 谓 的 “ 水 位 线 生 成 策 略 ” 。 WatermarkStrategy 中 包 含 了 一 个 “ 时 间 戳 分 配 器”TimestampAssigner 和一个“水位线生成器” WatermarkGenerator。

```java
public interface WatermarkStrategy<T> extends TimestampAssignerSupplier<T>,
    WatermarkGeneratorSupplier<T> {
    @Override
    TimestampAssigner<T> createTimestampAssigner(
        TimestampAssignerSupplier.Context context);

    @Override
    WatermarkGenerator<T> createWatermarkGenerator(
        WatermarkGeneratorSupplier.Context context);
}
```

#### 3. Flink 内置水位线生成器

WatermarkStrategy 这个接口是一个生成水位线策略的抽象，让我们可以灵活地实现自己的 需求；但看起来有些复杂，如果想要自己实现应该还是比较麻烦的。好在 Flink 充分考虑到了 我们的痛苦，提供了内置的水位线生成器（WatermarkGenerator），不仅开箱即用简化了编程， 而且也为我们自定义水位线策略提供了模板。

这两个生成器可以通过调用 WatermarkStrategy 的静态辅助方法来创建。它们都是周期性生成水位线的，分别对应着处理有序流和乱序流的场景。

（1）有序流

对于有序流，主要特点就是时间戳单调增长（Monotonously Increasing Timestamps），所以永远不会出现迟到数据的问题。 这是周期性生成水位线的最简单的场景， 直接调用 WatermarkStrategy.forMonotonousTimestamps()方法就可以实现。简单来说，就是直接拿当前最 大的时间戳作为水位线就可以了。

```java
stream.assignTimestampsAndWatermarks( WatermarkStrategy<Event>forMonotonousTimestamps().withTimestampAssigner(new SerializableTimestampAssigner<Event>(){

@Override
public long extractTimestamp(Event element, long recordTimestamp){
	return element.timestamp;
}
}));
```

上面代码中我们调用.withTimestampAssigner()方法，将数据中的 timestamp 字段提取出来， 作为时间戳分配给数据元素；然后用内置的有序流水位线生成器构造出了生成策略。这样，提 取出的数据时间戳，就是我们处理计算的事件时间。

这里需要注意的是，时间戳和水位线的单位，必须都是毫秒。

（2）乱序流

由于乱序流中需要等待迟到数据到齐， 所以必须设置一个固定量的延迟时间（ Fixed Amount of Lateness）。这时生成水位线的时间戳，就是当前数据流中最大的时间戳减去延迟的 结果， 相当于把表调慢， 当前时钟会滞后于数据的最大时间戳。 调用 WatermarkStrategy. forBoundedOutOfOrderness()方法就可以实现。这个方法需要传入一个 maxOutOfOrderness 参 数，表示“最大乱序程度”，它表示数据流中乱序数据时间戳的最大差值；如果我们能确定乱序 程度，那么设置对应时间长度的延迟，就可以等到所有的乱序数据了。

代码示例如下：

```java
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.time.Duration;


public class WatermarkTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        env.addSource(new ClickSource()).assignTimestampsAndWatermarks(// 针对乱序流插入水位线，延迟时间设置为 5s
        WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                                                                                        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
                // 抽取时间戳的逻辑
                @Override
                public long extractTimestamp(Event element, long recordTimestamp) {
                    return element.timestamp;
                }
            })).print();
        env.execute();
    }
}

```

上面代码中，我们同样提取了 timestamp 字段作为时间戳，并且以 5 秒的延迟时间创建了 处理乱序流的水位线生成器。

事实上，有序流的水位线生成器本质上和乱序流是一样的，相当于延迟设为 0 的乱序流水 位线生成器，两者完全等同：

```java
WatermarkStrategy.forMonotonousTimestamps()
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(0))
```

这里需要注意的是，乱序流中生成的水位线真正的时间戳，其实是 当前最大时间戳 – 延 迟时间 – 1，这里的单位是毫秒。为什么要减 1 毫秒呢？我们可以回想一下水位线的特点：时 间戳为 t 的水位线，表示时间戳≤t 的数据全部到齐，不会再来了。如果考虑有序流，也就是 延迟时间为 0 的情况，那么时间戳为 7 秒的数据到来时，之后其实是还有可能继续来 7 秒的数 据的；所以生成的水位线不是 7 秒，而是 6 秒 999 毫秒，7 秒的数据还可以继续来。这一点可 以在 BoundedOutOfOrdernessWatermarks 的源码中明显地看到：

```java
public void onPeriodicEmit(WatermarkOutput output) {
	output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis - 1));
}
```

#### 4. 自定义水位线策略

一般来说，Flink 内置的水位线生成器就可以满足应用需求了。不过有时我们的业务逻辑 可能非常复杂，这时对水位线生成的逻辑也有更高的要求，我们就必须自定义实现水位线策略 WatermarkStrategy 了。

在 WatermarkStrategy 中，时间戳分配器 TimestampAssigner 都是大同小异的，指定字段提 取时间戳就可以了；而不同策略的关键就在于 WatermarkGenerator 的实现。整体说来，Flink 有两种不同的生成水位线的方式：一种是周期性的（Periodic），另一种是断点式的（Punctuated）。

还记得 WatermarkGenerator 接口中的两个方法吗？——onEvent()和 onPeriodicEmit()，前 者是在每个事件到来时调用，而后者由框架周期性调用。周期性调用的方法中发出水位线，自 然就是周期性生成水位线；而在事件触发的方法中发出水位线，自然就是断点式生成了。两种 方式的不同就集中体现在这两个方法的实现上。

（1）周期性水位线生成器（Periodic Generator）

周期性生成器一般是通过 onEvent()观察判断输入的事件，而在 onPeriodicEmit()里发出水 位线。

下面是一段自定义周期性生成水位线的代码：

```java
import org.apache.flink.api.common.eventtime.*;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;


// 自定义水位线的产生
public class CustomWatermarkTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        env.addSource(new ClickSource())
           .assignTimestampsAndWatermarks(new CustomWatermarkStrategy()).print();

        env.execute();
    }

    public static class CustomWatermarkStrategy implements WatermarkStrategy<Event> {
        @Override
        public TimestampAssigner<Event> createTimestampAssigner(
            TimestampAssignerSupplier.Context context) {
            return new SerializableTimestampAssigner<Event>() {
                    @Override
                    public long extractTimestamp(Event element,
                        long recordTimestamp) {
                        return element.timestamp; // 告诉程序数据源里的时间戳是哪一个字段
                    }
                };
        }

        @Override
        public WatermarkGenerator<Event> createWatermarkGenerator(
            WatermarkGeneratorSupplier.Context context) {
            return new CustomPeriodicGenerator();
        }
    }

    public static class CustomPeriodicGenerator implements WatermarkGenerator<Event> {
        private Long delayTime = 5000L; // 延迟时间
        private Long maxTs = Long.MIN_VALUE + delayTime + 1L; // 观察到的最大时间戳

        @Override
        public void onEvent(Event event, long eventTimestamp,
            WatermarkOutput output) {
            // 每来一条数据就调用一次
            maxTs = Math.max(event.timestamp, maxTs); // 更新最大时间戳
        }

        @Override
        public void onPeriodicEmit(WatermarkOutput output) {
            // 发射水位线，默认 200ms 调用一次
            output.emitWatermark(new Watermark(maxTs - delayTime - 1L));
        }
    }
}

```

我们在 onPeriodicEmit()里调用 output.emitWatermark()，就可以发出水位线了；这个方法 由系统框架周期性地调用，默认 200ms 一次。所以水位线的时间戳是依赖当前已有数据的最 大时间戳的（这里的实现与内置生成器类似，也是减去延迟时间再减 1），但具体什么时候生 成与数据无关。

（2）断点式水位线生成器（Punctuated Generator）

断点式生成器会不停地检测 onEvent()中的事件，当发现带有水位线信息的特殊事件时， 就立即发出水位线。一般来说，断点式生成器不会通过 onPeriodicEmit()发出水位线。

自定义的断点式水位线生成器代码如下：

```java
public class CustomPunctuatedGenerator implements WatermarkGenerator<Event> {
    @Override
    public void onEvent(Event r, long eventTimestamp, WatermarkOutput output) {
        // 只有在遇到特定的 itemId 时，才发出水位线
        if (r.user.equals("Mary")) {
            output.emitWatermark(new Watermark(r.timestamp - 1));
        }
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // 不需要做任何事情，因为我们在 onEvent 方法中发射了水位线
    }
}

```

我们在 onEvent()中判断当前事件的 user 字段，只有遇到“Mary”这个特殊的值时，才调用 output.emitWatermark()发出水位线。这个过程是完全依靠事件来触发的，所以水位线的生成一 定在某个数据到来之后。

#### 5. 在自定义数据源中发送水位线

我们也可以在自定义的数据源中抽取事件时间，然后发送水位线。这里要注意的是，在自 定义数据源中发送了水位线以后，就不能再在程序中使用 assignTimestampsAndWatermarks 方 法 来生 成水 位线 了。 在自 定义 数据 源中 生成 水位 线和 在程 序中 使用 assignTimestampsAndWatermarks 方法生成水位线二者只能取其一。示例程序如下：

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.watermark.Watermark;

import java.util.Calendar;
import java.util.Random;


public class EmitWatermarkInSourceFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env
             = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(1);

        env.addSource(new ClickSourceWithWatermark()).print();

        env.execute();
    }

    // 泛型是数据源中的类型
    public static class ClickSourceWithWatermark implements SourceFunction<Event> {
        private boolean running = true;

        @Override
        public void run(SourceContext<Event> sourceContext)
            throws Exception {
            Random random = new Random();

            String[] userArr = { "Mary", "Bob", "Alice" };

            String[] urlArr = { "./home", "./cart", "./prod?id=1" };

            while (running) {
                long currTs = Calendar.getInstance().getTimeInMillis(); // 毫秒时间戳

                String username = userArr[random.nextInt(userArr.length)];

                String url
                     = urlArr[random.nextInt(urlArr.length)];

                Event event = new Event(username, url, currTs);

                // 使用 collectWithTimestamp 方法将数据发送出去，并指明数据中的时间戳的字段
                sourceContext.collectWithTimestamp(event, event.timestamp);

                // 发送水位线
                sourceContext.emitWatermark(new Watermark(event.timestamp - 1L));

                Thread.sleep(1000L);
            }
        }

        @Override
        public void cancel() {
            running = false;
        }
    }
}
```

在自定义水位线中生成水位线相比 assignTimestampsAndWatermarks 方法更加灵活，可以 任意的产生周期性的、非周期性的水位线，以及水位线的大小也完全由我们自定义。所以非常 适合用来编写 Flink 的测试程序，测试 Flink 的各种各样的特性。

### 水位线的传递

我们知道水位线是数据流中插入的一个标记，用来表示事件时间的进展，它会随着数据一 起在任务间传递。如果只是直通式（forward）的传输，那很简单，数据和水位线都是按照本 身的顺序依次传递、依次处理的；一旦水位线到达了算子任务, 那么这个任务就会将它内部的 时钟设为这个水位线的时间戳。

在这里，“任务的时钟”其实仍然是各自为政的，并没有统一的时钟。实际应用中往往上 下游都有多个并行子任务，为了统一推进事件时间的进展，我们要求上游任务处理完水位线、 时钟改变之后，要把当前的水位线再次发出，广播给所有的下游子任务。这样，后续任务就不 需要依赖原始数据中的时间戳（经过转化处理后，数据可能已经改变了），也可以知道当前事件时间了。

!!! question "思考"

    可是还有另外一个问题，那就是在“重分区”（redistributing）的传输模式下，一个任务有可能会收到来自不同分区上游子任务的数据。而不同分区的子任务时钟并不同步，所以同一时刻发给下游任务的水位线可能并不相同。这时下游任务又该听谁的呢？

这就要回到水位线定义的本质了：它表示的是“当前时间之前的数据，都已经到齐了”。 这是一种保证，告诉下游任务“只要你接到这个水位线，就代表之后我不会再给你发更早的数 据了，你可以放心做统计计算而不会遗漏数据”。所以如果一个任务收到了来自上游并行任务 的不同的水位线，说明上游各个分区处理得有快有慢，进度各不相同比如上游有两个并行子任 务都发来了水位线，一个是 5 秒，一个是 7 秒；这代表第一个并行任务已经处理完 5 秒之前的 所有数据，而第二个并行任务处理到了 7 秒。那这时自己的时钟怎么确定呢？当然也要以“这 之前的数据全部到齐”为标准。如果我们以较大的水位线 7 秒作为当前时间，那就表示“7 秒 前的数据都已经处理完”，这显然不是事实——第一个上游分区才处理到 5 秒，5~7 秒的数据 还会不停地发来；而如果以最小的水位线 5 秒作为当前时钟就不会有这个问题了，因为确实所 有上游分区都已经处理完，不会再发 5 秒前的数据了。这让我们想到“木桶原理”：所有的上 游并行任务就像围成木桶的一块块木板，它们中最短的那一块，决定了我们桶中的水位。

<figure markdown>
  ![image-20230210200532885](https://cos.gump.cloud/uPic/image-20230210200532885.png)
  <figcaption>任务间的水位线传递</figcaption>
</figure>

我们可以用一个具体的例子，将水位线在任务间传递的过程完整梳理一遍。如图所 示，当前任务的上游，有四个并行子任务，所以会接收到来自四个分区的水位线；而下游有三 个并行子任务，所以会向三个分区发出水位线。具体过程如下：

（1）上游并行子任务发来不同的水位线，当前任务会为每一个分区设置一个“分区水位线” （Partition Watermark），这是一个分区时钟；而当前任务自己的时钟，就是所有分区时钟里最 小的那个。

（2）当有一个新的水位线（第一分区的 4）从上游传来时，当前任务会首先更新对应的分 区时钟；然后再次判断所有分区时钟中的最小值，如果比之前大，说明事件时间有了进展，当 前任务的时钟也就可以更新了。这里要注意，更新后的任务时钟，并不一定是新来的那个分区 水位线，比如这里改变的是第一分区的时钟，但最小的分区时钟是第三分区的 3，于是当前任 务时钟就推进到了 3。当时钟有进展时，当前任务就会将自己的时钟以水位线的形式，广播给 下游所有子任务。

（3）再次收到新的水位线（第二分区的 7）后，执行同样的处理流程。首先将第二个分区 时钟更新为 7，然后比较所有分区时钟；发现最小值没有变化，那么当前任务的时钟也不变，也不会向下游任务发出水位线。 （4）同样道理，当又一次收到新的水位线（第三分区的 6）之后，第三个分区时钟更新为 6，同时所有分区时钟最小值变成了第一分区的 4，所以当前任务的时钟推进到 4，并发出时间 戳为 4 的水位线，广播到下游各个分区任务。

水位线在上下游任务之间的传递，非常巧妙地避免了分布式系统中没有统一时钟的问题， 每个任务都以“处理完之前所有数据”为标准来确定自己的时钟，就可以保证窗口处理的结果 总是正确的。对于有多条流合并之后进行处理的场景，水位线传递的规则是类似的。

### 水位线的总结

水位线在事件时间的世界里面，承担了时钟的角色。也就是说在事件时间的流中，水位线 是唯一的时间尺度。如果想要知道现在几点，就要看水位线的大小。后面讲到的窗口的闭合， 以及定时器的触发都要通过判断水位线的大小来决定是否触发。

水位线是一种特殊的事件，由程序员通过编程插入的数据流里面，然后跟随数据流向下游 流动。

水位线的默认计算公式：水位线 = 观察到的最大事件时间 – 最大延迟时间 – 1 毫秒。

所以这里涉及到一个问题，就是不同的算子看到的水位线的大小可能是不一样的。因为下游的算子可能并未接收到来自上游算子的水位线，导致下游算子的时钟要落后于上游算子的时钟。比如 map->reduce 这样的操作，如果在 map 中编写了非常耗时间的代码，将会阻塞水位线的向下传播，因为水位线也是数据流中的一个事件，位于水位线前面的数据如果没有处理完毕，那么水位线不可能弯道超车绕过前面的数据向下游传播，也就是说会被前面的数据阻塞。 这样就会影响到下游算子的聚合计算，因为下游算子中无论由窗口聚合还是定时器的操作，都需要水位线才能触发执行。这也就告诉了我们，在编写 Flink 程序时，一定要谨慎的编写每一个算子的计算逻辑，尽量避免大量计算或者是大量的 IO 操作，这样才不会阻塞水位线的向下 传递。

在数据流开始之前，Flink 会插入一个大小是负无穷大（在 Java 中是-Long.MAX_VALUE） 的水位线，而在数据流结束时，Flink 会插入一个正无穷大(Long.MAX_VALUE)的水位线，保 证所有的窗口闭合以及所有的定时器都被触发。

对于离线数据集，Flink 也会将其作为流读入，也就是一条数据一条数据的读取。在这种情况下，Flink 对于离线数据集，只会插入两次水位线，也就是在最开始处插入负无穷大的水位线，在结束位置插入一个正无穷大的水位线。因为只需要插入两次水位线，就可以保证计算 的正确，无需在数据流的中间插入水位线了。

水位线的重要性在于它的逻辑时钟特性，而逻辑时钟这个概念可以说是分布式系统里面最为重要的概念之一了，理解透彻了对理解各种分布式系统非常有帮助。具体可以参考 Leslie Lamport 的论文。
