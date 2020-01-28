### ContextCleaner

ContextCleaner上下文清理器扮演着Spark Core中垃圾收集器的角色，emmmmm......可以类比Java GC。它是在SparkContext中进行初始化的，代码非常简单，只依赖于
SparkContext本身，由spark.cleaner.referenceTracking配置项控制是否启用，默认值是启用状态。

下面分析一下ContextCleaner类的主要成员变量：
  * referenceBuffer：缓存CleanupTaskWeakReference的集合，CleanupTaskWeakReference是Java自带WeakReference类的简单封装，保存有需要清理的Spark组件实例的弱引用，当
  其中的reference对象可达性变为弱可达时，对应的CleanupTaskWeakReference实例就会被加入到ReferenceQueue中;

  * referenceQueue：缓存弱引用实例的引用队列(java.lang.ref.ReferenceQueue类型)对弱引用和软引用实例，当其被GC之后就会存入引用队列中，用户程序通过从队列中取得这些引用信
  息，就可以执行自定义的清理操作;

  * listeners：ContextCleaner的监听器队列，目前只是在测试代码中用到;

  * cleaningThread：执行具体清理工作的线程，具体是调用了keepCleaning()方法;

  * periodicGCService：一个单线程的调度线程池，用来周期性地执行GC操作;

  * periodicGCInterval：periodicGCService执行GC的周期长度，由spark.cleaner.periodicGC.interval配置项控制，默认30分钟;

  * blockOnCleanupTasks：执行清理任务的时候是否阻塞(不包含Shuffle数据的清理任务)，由spark.cleaner.referenceTracking.blocking配置项控制，默认阻塞;

  * blockOnShuffleCleanupTasks：执行清理Shuffle数据的任务时是否阻塞，由spark.cleaner.referenceTracking.blocking.shuffle配置项控制，默认不阻塞;

  * stopped：该ContextCleaner是否停止的标记;

ContextCleaner中共有五种清理任务，分别对应RDD、Shuffle、广播变量、累加器和检查点，都继承于CleanupTask这个空特征，清理RDD调用SparkContext.unpersistRDD()
方法来反持久化RDD，清理Shuffle则需要同时从MapOutputTracker与BlockManager中反注册Shuffle。清理完毕后再调用各个监听器的监听方法进行记录。其中提供的成员方法如下：
  * start()：将清理线程cleaningThread设置为守护线程并启动，然后按照periodicGCInterval的间隔来调度执行System.gc()方法，从而可能触发GC。
  所以在Spark Application指定Driver和Executor的JVM启动参数时，不能加上-XX:-DisableExplicitGC，因为该参数会让System.gc()的调用无效;

  * registerCleaner()：用于将CleanupTask及其对应的要清理的对象加入到referenceBuffer集合中;

  * keepCleaning()：从ReferenceQueue中取出CleanupTaskWeakReference，然后将其包含的CleanupTask进行模式匹配，并对五种情况分别对应调用不同的方法;