### SparkContext其它相关

在[Spark源码阅读2：SparkContext主要组件](../master/docs/sparkcontext-components.md)中已经熟悉了SparkContext的组件初始化部分，它们也是SparkContext
的主体，除此之外，SparkContext中还有一些与其内部机制紧密关联的属性和完成了许多其它的辅助工作(毕竟共有大概3100行代码呢!!!)，对于它们也有必要做一个大致的了解。

辅助属性：
  * creationSite：CallSite类型，用于指示这个SparkContext是在哪里创建的。CallSite是一个简单的数据结构，用来描述用户代码的一个位置，拥有两个属性：shortForm
  和longForm。SparkContext使用Utils.getCallSite()方法遍历当前线程的线程栈，并找到一个最靠近栈顶的Spark方法调用和最靠近栈底的用户方法调用，将它们以短格式和
  长格式包装在CallSite中并返回。

  * startTime & stopped：前者指示SparkContext的启动时间戳，后者使用AtomicBoolean指示SparkContext是否已停止。

  * addedFiles/addedJars & _files/_jars：用户自定义的文件与jar包。前两者是ConcurrentHashMap类型，ConcurrentHashMap的键存储文件及jar包的url，
  ConcurrentHashMap值是文件被加入当时的时间戳。后两者是Spark配置文件中定义的文件或jar包路径，Seq类型，表示从配置文件的配置项中取出路径组成的序列，然后再循环
  序列将路径及添加事件放到前者的Map中。

  * persistentRdds：维护持久化RDD的ID与其弱引用的映射关系，可以通过cache()/persist()持久化RDD，通过unpersist()方法反持久化RDD，最终分别调用的是
  SparkContext.persistRDD()和SparkContext.unpersistRDD()方法。

  * executorEnvs & _executorMemory & sparkUser：executorEnvs是一个HashMap，Driver向Executor分配任务时可能需要传递一些环境变量，executorEnvs就用来
  存储需要传递的环境变量，而_executorMemory和sparkUser包含在其中，分别代表Executor的内存大小和当前运行SparkContext的用户名。Executor内存可以通过
  spark.executor.memory配置项、SPARK_EXECUTOR_MEMORY环境变量、SPARK_MEM环境变量指定，优先级从高到低，默认1024MB。用户名则是通过Utils.getCurrentUserName()
  方法获得。

  * checkpointDir：指定在集群环境下，RDD的检查点在HDFS上保存的目录，这样当计算过程中发生错误时能够快速进行恢复，而不需要重新进行计算，可以通过setCheckpointDir()
  方法进行设置。

  * localProperties：维护了一个Properties类型的线程本地变量，它是InheritableThreadLocal类型，继承自ThreadLocal，在ThreadLocal的基础上允许本地变量从父线程到
  子线程的继承，也就是Properties会沿着线程栈传递。

  * _eventLogDir & _eventLogCodec：都与EventLoggingListener有关，当spark.eventLog.enabled为true时，会将事件日志写入_eventLogDir指定的目录，通过
  spark.eventLog.dir设置。_eventLogCodec则是指定了事件日志的压缩算法，如果spark.eventLog.compress为true则根据spark.eventLog.compression.codec设置的压缩
  算法进行压缩。

  * _applicationId & _applicationAttemptId：都是TaskScheduler初始化并启动后分配，只有在TaskScheduler启动后，应用代码逻辑才会真正被执行，且由于各种异常原因
  进行多次的重试。

  * _shutdownHookRef：定义SparkContext的关闭钩子，当JVM退出时，显示的执行SparkContext.stop()方法，以防用户忘记而未执行。

  * nextShuffleId & nextRddId：都是AtomicInteger类型，Shuffle和RDD都需要唯一ID进行标识，且都是递增的。

其它初始化方法：
  * setupAndStartListenerBus()：注册自定义的监听器，并启动LiveListenerBus。自定义监听器实现了SparkListener特征，并通过spark.extraListeners指定。通过
  Utils.loadExtensions()方法通过反射来构建自定义监听器实例，并注册到LiveListenerBus。

  * postEnvironmentUpdate()：在添加自定义文件和jar包时也需要被调用，它取得当前自定义文件和jar包列表，以及Spark的配置、调度方式，然后通过
  SparkEnv.environmentDetails()方法取得JVM参数、Java系统属性等，封装成SparkListenerEnvironmentUpdate事件投递给事件总线。

  * postApplicationStart()方法：向事件总线投递SparkListenerApplicationStart事件，表示应用已启动。

通用功能：
  * 生成RDD：相信对Spark有过一定了解的都知道生成RDD有三种办法，一是对内存中已经存在的数据进行并行化(makeRDD、parallelize)操作，二是从外部数据源中读取数据，三是
  从其它RDD进行转换，其中前两种方法都在SparkContext中。第一种方法都是生成了ParallelCollectionRDD类型的RDD，可以给parallelize()方法传递一个参数表示其默认的分
  区数，也就是其Task的默认并行度。而第二种办法可以调用的方法比较多，这里以最简单的textFile()为例，它可以使用TextInputFormat格式读取HDFS上指定路径的文件，生成的
  是存储二元组的HadoopRDD，再将其中的内容用map()算子提取出来，textFile也可以传递默认分区数，但是值得注意的是parallelize()传递的是默认分区数，而textFile()传递
  的是最小分区数，也就是说textFile()实际产生的分区数可能比传入的分区数大。

  * 广播变量：Spark中有两种共享变量，广播变量就是其中一种(还有一种是下面即将要说到的累加器变量)。广播是指Driver直接向每个Worker节点发送同一份数据的只读副本，而不
  经过Task的计算，比较适用于处理多节点跨Stage的共享数据和配置等，典型场景是大集合与小集合的join，为了避免shuffle，将数据量较小的RDD广播以避免大数据量shuffle带来的性能
  开销甚至是OOM异常。

  * 累加器：累加器与广播变量一样也是Spark的共享变量，其就是一个能够累计结果值的变量，计数器就是其常见用途。为了能够保持唯一性，它只在Driver端创建和读取，Executor
  端只能进行累加操作，SparkContext已经提供了数值型累加器的创建方法(如LongAccumulator)，AccumulatorV2抽象类是所有累加器的基类，当然了，我们也可以自定义其它类型
  的累加器，AccumulatorMetadata用于封装累加器对应的数据类型及累加操作。

  * 启动Job：Spark提供了多种runJob()方法的重载来运行job，也就是触发RDD动作算子的执行，但是其实最终都是调用了DAGScheduler.runJob()来运行job，这个方法会将需要
  计算的RDD及其分区列表传入，运行时会对RDD保存检查点，计算完成后再将结果回传给resultHandler回调方法。

后启动方法：
  * TaskScheduler.postStartHook()方法，主要用于Spark环境下当应用成功初始化后调用，YARN模式下会使用它去基于优先位置启动资源的分配，并等待从节点的注册。

  * 在Metrics系统中注册DAGScheduler、BlockManager、ExecutionAllocationManager的Metrics source，以便收集相应的监控数据。

  * 添加shutdownHook，这个在上面有讲到，不再说。

SparkContext中有很多伴生对象，它们辅助SparkContext完成了很多功能，createTaskScheduler()方法就来自于SparkContext的伴生对象，除它之外，其它的伴生对象主要用来
跟踪并维护SparkContext的创建与激活。SparkContext的伴生对象中有三个属性与SparkContext的创建过程相关，SPARK_CONTEXT_CONSTRUCTOR_LOCK是SparkContext构造过程
中使用的锁对象，用于保证线程安全。activeContext用于保存当前活动的SparkContext的原子引用。contextBeingContructed用于保证当前正在创建的SparkContext。
  * markPartiallyConstructed(this)，将当前SparkContext标记为正在创建，使用assertNoOtherContextRunning()私有方法检测是否有多个SparkContext
  实例正在运行，在有另一个SparkContext创建时打印警告，当有一个SparkContext运行时抛出异常，从而阻止同一时间存在多个活动的SparkContexts。在早期的版本中是允许多个
  活动的SparkContexts的，但是在[SPARK-26362]已移除该支持，不再允许同时存在多个活动的SparkContexts。

  * getOrCreate()方法，是创建SparkContext的另一种方式，相对于new SparkContext()，它会先检查当前有没有已经激活的SparkContext，如果有则直接复用，没有再创建。

  * 最后的最后，setActiveContext()方法，在SparkContext的主构造方法的最末尾处调用，将当前SparkContext设为活动的。

至此，SparkContext的部分终于已经了解得差不多了。