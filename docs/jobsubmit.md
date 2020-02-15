### Spark任务提交

在用户应用程序的代码中，不论是调用saveAsHadoopDataset()方法还是调用其他的方法启动Spark任务，触发action操作后，最终都会调用SparkContext中的runJob()方法。
SparkContext在[Spark源码阅读2：SparkContext主要组件](./sparkcontextcomponents.md)和[Spark源码阅读3：SparkContext其它相关](./sparkcontextother.md)
中进行了大量的分析，所以不再赘述，直接来看runJob()这个方法吧，该方法在RDD的所有分区上运行任务，并且以数据的方式返回结果，它是Spark里所有action的主入口。它内部是
调用dagScheduler的runJob()方法执行，这个方法中通过submitJob()方法进行任务的提交，并返回JobWaiter对象，该对象会等待job的执行完成，然后传递所有的结果到
resultHandler()方法进行后续的处理。

调用关系如下：
  1. org.apache.spark.SparkContext#runJob()

  2. org.apache.spark.scheduler.DAGScheduler#runJob()

  3. org.apache.spark.scheduler.DAGScheduler#submitJob()

  4. org.apache.spark.scheduler.DAGSchedulerEventProcessLoop#onReceive(JobSubmitted)

  5. org.apache.spark.scheduler.DAGSchedulerEventProcessLoop#doOnReceive(JobSubmitted)

  6. org.apache.spark.scheduler.DAGScheduler#handleJobSubmitted()

在DAGScheduler.submitJob()方法中进行提交前，会先进行必要的检查以确保没有在一个不存在的分区上提交任务。在创建jobId后创建JobWaiter对象，并使用类型为DAGSchedulerEventProcessLoop
的对象eventProcessLoop，将任务提交JobSubmitted对象放置在event队列中，eventThread后台线程将对任务提交进行处理，这个eventThread被定义在DAGSchedulerEventProcessLoop
的父类EventLoop当中。EventLoop就是从调用者接收事件并且启动给一个额外的eventThread对所有在eventThread中的事件进行处理。(值得注意的是在EventLoop中，事件队列
eventQueue是一个LinkedBlockingDeque可能会无限制的增长，因此子类在继承EventLoop时必须实现onReceive()方法并确保onReceive()能够及时的处理事件以避免可能的OOM错误.)

DAGSchedulerEventProcessLoop就是EventLoop的子类，它对onReceive()进行了实现，主要是在其中调用了doOnReceive()方法，在doOnReceive()方法的具体实现中，
会根据事件类型调用对应的方法进行处理，此处是JobSubmitted类型事件，所以会调用dagScheduler的handlerJobSubmitted方法完成整个job的提交。

在handleJobSubmitted()方法中，主要的逻辑是：
  * 首先会根据finalRDD调用DAGScheduler.createResultStage()创建finalStage，并划分Stage。finalStage就是最后的那个Stage，它是所有Spark程序一定会有的那个Stage，Spark程序的Stage数量
  为一个finalStage再加上宽依赖的个数。finalStage包含的是一组ResultTask，它们会将计算结果发送回Driver Application。此外，还有一种ShuffleMapTask，它根据Task的partitioner
  将计算结果放到不同的bucket中。Stage又是什么呢？在[Spark源码阅读2：SparkContext主要组件](./sparkcontextcomponents.md)中已经说过，Task是集群上运行的基本单位，一个Task会
  负责处理RDD的一个Partition。所以RDD的多个partition会分别由不同的Task去处理(虽然这些Task的处理逻辑完全是一样的)，这一组Task就组成了一个Stage，一个Stage的开始就是从外部存储
  或者shuffle结果中读取数据，一个Stage的结束就是由于发生shuffle或者生成结果时，而一个Job则包含多个Stage;

  * 根据finalStage的类型创建job，如果finalStage是ResultStage则创建一个resultJob，否则如果是ShuffleMapStage则创建一个mapStageJob;

  * 向LiveListenerBus提交一个SparkListenerJobStart事件，listenerThread后台线程会处理该事件;

  * 最后，提交finalStage，该方法会提交所有关联的未提交的stage;