### LiveListenerBus事件总线

事件总线LiveListenerBus是Spark中非常重要的组件，前面已经提到它采用了监听器的设计模式，其大致流程如下图：

![LiveListenerBus事件总线](../image/livelistenerbus.png "LiveListenerBus事件总线")

ListenerBus特征是Spark内所有事件总线实现的基类，下图列出了它的一部分继承体系：

![ListenerBus事件总线](../image/listenerbus.png "ListenerBus继承体系")

StreamingListenerBus和StreamingQueryListenerBus分别是Spark Streaming和Spark SQL的组件，这里主要看ListenerBus和SparkListenerBus这两个trait。
ListenerBus带有两个泛型参数L和E，其中L表示监听器类型，它可以是任意类型的引用，E表示的是事件类型。而其属性listenerPlusTimers则维护了所有注册在事件总线
上的监听器及对应计时器的二元元组，Option表示计时器是可选的，用来指示监听器处理事件的时间，采用并发容器CopyOnWriteArrayList支持并发的修改以保证线程安全。
而属性listeners根据代码可以看出，是将listenersPlusTimers中的监听器取出并转化为java中的List类型。

ListenerBus中还定义了一些与事件总线相关的方法，以下简单的说一下：
  * addListener() & removeListener()方法：分别向事件总线添加事件监听器和移除事件监听器，实现上很简单，就是在并发容器CopyOnWriteArrayList上执行add()
  方法和remove()方法。
  * doPostEvent()方法：将事件event投递给监听器listener进行处理，在此处只进行了抽象定义，具体逻辑在其实现类中。
  * postToAll()方法：通过doPostEvent()方法，将事件event投递给所有已经注册的监听器，由于它是线程不安全的，因此同时只能被一个线程调用。

SparkListenerBus这个trait是Spark Core内部事件总线的基类，继承于ListenerBus并实现了doPostEvent()方法来对事件进行匹配，并调用监听器的处理方法。如果无法
匹配到事件，则调用onOtherEvent()方法，它所支持的监听器都是SparkListenerInterface的子类，事件则是SparkListenerEvent的子类。其中，SparkListenerInterface
定义了每个事件的处理方法，命名规则是"onXXX"，而SparkListenerEvent则是一个没有任何抽象方法的trait，它唯一的用途是以"SparkListener+事件名称"标记具体的事件
类。

SparkListenerBus默认提供的事件投递方法是同步调用的，如果注册的监听器和产生的事件很多，同步调用必然带来事件的积压和处理的延时，因此在SparkListenerBus的
实现类AsyncEventQueue中，提供了异步事件队列机制，它也是SparkContext中的事件总线LiveListenerBus的基础。

AsyncEventQueue，异步事件队列，一看这个名字就很异步，类的注释上也写得很明白，所有投递到这个队列的事件都会被一个单独的线程分发到所有子监听器上去。它的构造参数
有四个，分别是队列的名称、配置项、LiveListenerBus的监控metrics和LiveListenerBus本身。先来看下它有哪些主要属性：
  * eventQueue：LinkedBlockingQueue类型，用来存储SparkListenerEvent事件的阻塞队列，其大小通过配置spark.scheduler.listenerbus.eventqueue.capacity
  控制，默认值10000，若不设置将会以Integer.MAX_VALUE为默认值，可能会发生OOM。

  * eventCount：AtomicLong型以保证修改的原子性，表示当前待处理事件的计数，由于事件从eventQueue中取出并不能保证事件已被处理完毕，故不能用eventQueue的实际
  大小表示。

  * droppedEventsCounter：也是AtomicLong类型，但默认值为0，表示被丢弃的事件数量，当eventQueue满后，新产生的事件无法入队就会被丢弃，会将该值定期的记录到
  日志中，且用lastReportTimestamp记录下最后一次写入该值到日志的时间戳，并将值重置为0。

  * lastReportTimestamp：上面已经提到，记录的是最后一次写入droppedEventsCounter到日志中的时间戳。

  * lastDroppedEvent：AtomicBoolean类型，用于表示是否发生过事件丢弃。

  * started：标记队列的启动状态，默认false，当start()方法调用以后就会变成true。

  * stopped：标记队列的停止状态，默认也为false，当stop()方法被调用时变成true。

  * dispatchThread：是将队列中的事件分发给所有子监听器的守护线程，实际上是重写了其run()方法并调用dispatch()方法。而Utils.tryOrStopSparkContext()方法
  的作用是如果执行代码的过程中发生异常，就另起一个线程关闭SparkContext。

再来看下SparkListenerBus的一些主要方法：
  * dispatch()：该方法循环的从eventQueue中取出事件，如果取出的事件不是POISON_PILL事件(这是在伴生对象中定义的一个特殊的空事件，当队列调用了stop()方法停止后
  放入，用于退出循环)，就调用父类ListenerBus的postToAll()方法将其投递给所有已注册的监听器，并减少eventCount的值。

  * post()：投递方法，即把事件投递到队列中的方法，它先检查队列是否已停止(即是否调用过stop()方法)，如果还没有停止，就增加eventCount的计数并将事件放入队列，若
  offer()方法返回true表示投递成功，若返回false，表示队列已满，将eventCount的计数减一，并标记有事件被丢弃，如果droppedEventsCounter值大于0，且系统当前时间
  戳与上次写入droppedEventsCounter到日志中的时间戳的间隔大于1分钟，则将droppedEventsCounter的值再次写入到日志中去。

最后来看一下SparkContext中已经提到过的一个重要的组件LiveListenerBus异步事件总线。上面已经讲到，AsyncEventQueue继承了SparkListenerBus特征，而
LiveListenerBus就是以AsyncEventQueue作为核心的。依然是先看一下它的主要属性，它的很多属性都已经在AsyncEventQueue中见到过，多出来的主要有queues和
queuedEvents这两个：
  * queues：CopyOnWriteArrayList类型以保证线程安全性，维护着AsyncEventQueue的列表，因此LiveListenerBus中会有多个事件队列。
  * queuedEvents：SparkListenerEvent的列表，用于在LiveListenerBus启动完成前缓存已经收到的事件，当启动完成后会优先的把这些事件投递出去。

再来看一下LiveListenerBus的主要方法：
  * addToQueue()：将监听器listener注册到名称为queue所标识的队列中，它需要在queues的列表中寻找符合条件的队列，如果队列存在就用父类ListenerBus的addListener()
  方法直接注册监听器，否则就创建一个AsyncEventQueue注册新监听器到新队列中，当然LiveListenerBus中还提供了其它4种注册监听器的方法，对应4个内置队列，分别表示共享队列、
  executor管理的队列、应用状态队列和事件日志队列。
  * post()、postToQueues()：post()方法会检查LiveListenerBus是否已经启动，以及queuedEvents中是否有缓存事件(未完全启动时产生的事件都会加到queuedEvents中)。投递时
  会直接调用postToQueues()方法，将事件发送给所有的队列，而由AsyncEventQueue实际完成投递到监听器的工作。

最后，用一张简图来表示AsyncEventQueue与LiveListenerBus的关系，并结束LiveListenerBus的源码研究。
![AsyncEventQueue与LiveListenerBus](../image/asynceventqueue.png "AsyncEventQueue与LiveListenerBus关系简图")