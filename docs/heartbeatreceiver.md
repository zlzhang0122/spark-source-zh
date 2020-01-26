### HeartbeatReceiver

在Spark中，Executor需要定期向Driver发送心跳以表明自己仍然存活，而HeartbeatReceiver是由Driver持有并负责处理各个Executor的心跳，监控它们的运行状态。

在代码实现上，HeartbeatReceiver类继承了SparkListener抽象类，并实现了ThreadSafeRpcEndpoint特征，这表明其既是一个监听器，也是一个线程安全的RPC端点。
HeartbeatReceiver类有两个构造方法参数，分别是SparkContext和Clock特征的实现类SystemClock类，SystemClock提供了对系统时间System.currentTimeMillis()
的简单封装。在HeartbeatReceiver构造时，会同时将其加入到LiveListenerBus的Executor管理队列(executorManagement)中进行监听。

下面简单介绍下HeartbeatReceiver中的部分成员属性：
  * executorLastSeen：维护Executor ID与收到该Executor最后一次心跳的时间戳之间的映射关系;

  * slaveTimeoutMs：由spark.storage.blockManagerSlaveTimeoutMs配置项控制，表示的是Executor上的BlockManager的超时时间，默认是120秒;

  * executorTimeoutMs：由配置项spark.network.timeout配置项控制，表示的是Executor自身的超时时间，默认值是spark.storage.blockManagerSlaveTimeoutMs
  配置的值;

  * timeoutIntervalMs：由配置项spark.storage.blockManagerTimeoutIntervalMs控制，表示的是检查Executor上BlockManager是否超时的时间间隔，默认是60秒;

  * checkTimeoutIntervalMs：由配置项spark.network.timeoutInterval控制，表示检查Executor是否超时的时间间隔，默认值是spark.storage.blockManagerTimeoutIntervalMs
  配置的值;

  * timeoutCheckingTask：持有检查Executor是否超时的任务返回的ScheduledFuture对象;

  * eventLoopThread：单守护线程的调度线程池，名称为heartbeat-receiver-event-loop-thread，是整个HeartbeatReceiver的事件处理线程;

  * killExecutorThread：单守护线程的普通线程池，名称为kill-executor-thread，用于异步执行kill Executor的任务;

介绍了HeartbeatReceiver类的这么多属性，那么再看看它都有哪些方法：
  * onStart()：HeartbeatReceiver是一个RPC端点，因此也实现了RpcEndpoint.onStart()方法，当RPC环境中的Dispatcher注册RPC端点时，将会调用该方法。
  这个方法会让eventLoopThread以spark.network.timeoutInterval规定的时间间隔调度执行，并将ScheduledFuture对象返回给timeoutCheckingTask。而
  eventLoopThread线程只做一件事情，那就是向HeartbeatReceiver自己发送ExpireDeadHosts消息，并等待回复。

