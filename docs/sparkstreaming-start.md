### SparkStreaming序

前面已经基本分析完了Spark Core的核心代码，用它来做离线计算已经完全够用了，但是咱的目的并不是这个啊，咱要学的是实时流式处理，嗯，这就是接下来一段
时间要分析的一个分支--Spark Streaming，它依赖于Spark Core作为基础架构，因此Spark Streaming会因为Spark Core本身的一些架构上的原因而有一些
天然的限制。比如说，Spark Streaming采用的是一种微批的处理方式，也就是它会将非常小的一段时间内的数据攒在一起形成一个微批，然后定时的去触发计算。
我们知道在Flink中有窗口的概念，一个窗口内的数据会在窗口结束时触发计算(或者用定时器也能触发)。那么同样都是攒批延迟计算，它们的区别在哪里呢？我觉得
从结果来看它们并没有多少区别，它们之间最大的区别就在于：Flink窗口的延迟计算是由业务导致的，也就是说业务上需要对乱序的数据进行处理，所以有了窗口；
而Spark Streaming的延迟计算是它本身的架构导致的，由于它依赖于Spark Core的架构，所以它只能用攒批的方式来实现准实时的流式计算。千万不要小看这个
区别，业务上的需求导致的问题可以随时改需求，但是系统架构的问题却不能随时改架构，所以看似导致的结果一样，但是差距上却是一条鸿沟。

架构上的缺陷注定了Spark Streaming更像是一种临时的解决方案，但是为何这种看上去很临时的解决方案却被广泛使用呢？因为Spark streaming架构在Spark之上，
而Spark出现的时间是比较早的，因此从Spark到Spark Streaming开发人员和企业都比较接受，运维也可以由同一拨人来完成，节省成本。这也说明了有时候对于市场
来说，并不一定是技术最好的产品最吃香，成本和效率往往更加关键。

说的有点跑题啦，言归正传。前面花了有好几个月将Spark Core的源码刚刚分析完，既然Spark Streaming是以Spark Core为基础架构来实现的，我们也知道其核心
原理就是微批，也就是将实时流式数据按一小段一小段的时间切分，变成极小的批，然后交给Spark去处理(用业界的话说，Spark是将流式计算看作是批处理的特例，而
Flink是将批处理看作是流式计算的特例)。

那么我们自己想一下，如果我们想要用微批的方式自己来实现用Spark进行流式计算，我们需要哪些条件？理解了这些条件，那么就很容易理解Spark Streaming！
  1. 首先，得有一块数据，然后就可以通过该块数据，利用RDD 的API构建出一个能够进行数据处理的RDD DAG(有向无环图);

  2. 其次，流式数据是持续不断的进入系统的，所以我们必须的源源不断的流式数据进行切片(比如100ms作为一个小批，将这个时间周期内的数据积攒成个小的数据块)，
  既然有了数据块，我们就能使用第一步的DAG图对每一个数据块的数据(微批)进行处理;

  3. 再次，由于数据是源源不断的接入，那么集邮可能在一个处理周期内(100ms)处理不完一块数据，那么同时存在多个块是必然的，也就是说同一时间存在多个DAG
  同时运行的情况(一个DAG图对应一小块数据)，这些小块可能存在相互依赖的关系，这些DAG图结构一样但处理着不同的数据属于不同的实例。

综上来看，我们需要的是：
  * 一个静态RDD DAG模版。表示对小块数据的处理逻辑;

  * 一个控制器。能够将连续的流式数据切分成小块的batch数据，并按照DAG模版为每一个小块数据生成新的DAG实例，对小块数据进行处理;

  * 原始数据的接入(类似于Storm的Spout，Flink的Source的组件);

  * 故障恢复机制。数据输入难免会有错误的格式，计算过程很难保证一定不会出现问题，如何在故障后自动重试和恢复非常重要，尤其是对于流式计算框架这种需要长
  时间7 * 24小时运行的系统而言。

好了，分析完我们想象的利用Spark Core作为底层架构来进行流式计算需要解决的三个问题及可能的四个组件，接下来分析SparkStreaming也主要是从四个层面来看Spark
Streaming是怎样提出针对性的解决方案的。

首先，是RDD DAG模版，Saprk Streaming根据该模版生成一个个的DAG实例。在Spark Streaming的世界里，这个DAG"模版"的具体实现就是DStreamGraph，而DStream
就类似于Spark Core中的RDD。在Spark Core中，RDD对应着很多的子类，几乎每一个子类都有一个对应的DStream，比如UnionRDD对应的是UnionDStream。RDD通过
转换算子连接成RDD DAG(有向无环图，但貌似没有具体的类)，而DStream也可以通过转换算子连接成DStreamGraph(对比一下，在Flink中叫做StreamGraph)。

那么，DStream和RDD的关系是怎么样的呢？既然DStream是RDD模版，而且都能进行转换操作(map、filter、reduce等)，那么它们有什么区别呢？首先，DStream维护着
每个产出的RDD实例的引用，其次，对于那些能够进行流控的DStream子类，还会记录用于进行流控的信息，如每次进行处理时源头数据的条数、计算所花费的时间等。因此，
我们可以将DStream理解为RDD+微批的维度数据。

值得注意的是，在Spark Stream和Flink中，DStream(Spark Streaming)和DataStream(Flink)是顶点，转换才是边(这与Storm中是完全相反的。Storm中，计算Spout
和Bolt是顶点，Tuple数据才是边)。

有了DStream和DStreamGraph，也即定义了数据的处理逻辑，再来看看Spark Streaming是如何进行动态调度的。
在Spark Streaming程序的入口，我们会定义一个batchDuration，它负责周期性的间隔一定的时间并比照静态的DStreamGraph动态生成RDD DAG实例。在Spark Streaming
中，负责动态调度作业的是JobScheduler类，在Spark Streaming程序开始运行前都会生成一个JobScheduler的实例，并通过start()运行起来。

JobScheduler有两个非常重要的成员：JobGenerator和ReceiverTracker。每个微批的RDD DAG的具体生成工作都是委托为JobGenerator的，而源头数据的记录工作则是委托
给了ReceiverTracker。

JobGenerator维护了一个定时器，周期就是前面有讲到的batchDuration，它定时的为每个微批生成RDD DAG的实例，每次的实际生成包含5个步骤：
  * 要求ReceiverTracker将自上一次微批切分后的到达的数据都切分到本次新的微批里，也就是对当前已收到但未处理的数据进行一次分配;

  * 要求DStreamGraph复制出一份新的RDD DAG实例，具体过程是：DStreamGraph将DAG图里末尾Stream节点生成具体的RDD实例，并递归的调用尾DStream的上游DStream
  节点，从而遍历整个DStreamGraph，遍历结束也就恰好生成了RDD DAG的实例;

  * 获取前面ReceiverTracker分配到本微批次的源头数据的meta息;

  * 将前面生成的RDD DAG和meta信息，共同提交给JobScheduler异步执行;

  * 提交结束时(不论是否已经开始执行)，立即对整个系统的当前运行状态做checkpoint;

