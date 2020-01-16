### NettyRpcEnv

NettyRpcEnv作为RPC环境，除了拥有接收消息的功能外，也应该能发送消息。那我们就先来看一看NettyRpcEnv中与消息发送相关的成员：
  * clientFactory、fileDownloadFactory：都是TransportClientFactory类型，通过transportContext.createClientFactory()方法创建，这个工厂类
  在NettyRpcEnv中用于生产TransportClient。clientFactory用于处理一般性的请求发送和应答，而fileDownloadFactory专用于下载文件，按需创建，不会立即
  被初始化。

  * timeoutScheduler：类型是ScheduledThreadPoolExecutor，也就是Java中的定时调度的线程池，通过ThreadUtils工具类中的newDaemonSingleThreadScheduledExecutor
  创建，默认只有一个守护线程，专门用于处理RPC请求的超时。

  * clientConnectionExecutor：类型是ThreadPoolExecutor，实际上是一个缓冲的守护线程池，专门用于处理TransportClient的创建，由于
  TransportClientFactory.createClient()方法本身就是一个阻塞调用，因此需要用线程池来进行异步处理，线程池大小由spark.rpc.connect.threads调节，
  默认大小是64。

  * outboxes：在[Spark源码阅读7：Dispatcher](../master/docs/dispatcher.md)中已经研究过Inbox这个"收件箱"组件了，outboxes就是与之对应的
  "发件箱"，维护着远端RPC地址与各个发件箱的映射，所有需要发送的消息都会先放入到Outbox中再进行处理，且里面的所有消息都继承自OutboxMessage特征。

OutboxMessage特征比较简单，其中只有两个方法：sendWith()和onFailure()。它有两个实现类，分别是无需应答的消息OneWayOutboxMessage和需要应答的消息
RpcOutboxMessage。

Outbox.send()方法真正用于发送消息，如果没有活跃的连接，则缓存待发送的消息并创建一个新的连接。如果Outbox已经停止，发送方会接收到一个SparkException异常。
如果不是停止状态，就将OutboxMessage添加到链表中，并调用drainOutbox()方法处理消息。它的messages与Inbox不一样，在这里的是一个普通链表，所以需要用
synchronized进行同步以保证线程安全。
