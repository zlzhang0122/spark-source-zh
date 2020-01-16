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

  * outboxes：在[Spark源码阅读7：Dispatcher](./master/docs/dispatcher.md)中已经研究过Inbox这个"收件箱"组件了，outboxes就是与之对应的
  "发件箱"，维护着远端RPC地址与各个发件箱的映射，所有需要发送的消息都会先放入到Outbox中再进行处理，且里面的所有消息都继承自OutboxMessage特征。

OutboxMessage特征比较简单，其中只有两个方法：sendWith()和onFailure()。它有两个实现类，分别是无需应答的消息OneWayOutboxMessage和需要应答的消息
RpcOutboxMessage。

Outbox.send()方法真正用于发送消息，如果没有活跃的连接，则缓存待发送的消息并创建一个新的连接(也就是将消息加入到messages中)。如果Outbox已经停止，发
送方会接收到一个SparkException异常。如果不是停止状态，就将OutboxMessage添加到链表中，并调用drainOutbox()方法处理消息。它的messages与Inbox不一
样，在这里的是一个普通链表，所以需要用synchronized进行同步以保证线程安全。

在drainOutbox()方法中，会进行判断，如果遇到一下三种情况，则不处理消息，直接返回：
  * Outbox已经停止，或者正处于连接远端RPC节点的过程当中；

  * TransportClient本身为空，说明需要先创建RPC客户端。在创建RPC客户端时，使用到了clientConnectionExecutor线程池来提交一个Callable，在其内部
  实现上会最终调用clientFactory.createClient()方法创建RPC客户端，创建成功后会再次调用drainOutbox()方法来处理消息。

  * 已经有其它线程正在处理消息。

如果没有任何异常情况，则直接从messages链表中取出消息，并标记draining为true表明消息正在被处理，然后调用OutboxMessage.sendWith()方法进行发送。

Outbox用于发送消息，但其中的消息是何时投递过来的呢？向其投递消息的逻辑位于NettyRpcEnv.postToOutbox()方法中，该方法的主要逻辑是，如果已经持有了
远端RPC端点引用对应的TransportClient，就直接调用OutboxMessage.sendWith()方法投递，否则就先从outboxes缓存中获取RPC地址对应的发件箱，如果发件箱
也没有就先new一个出来，最后判断如果NettyRpcEnv和Outbox没有停止的话，就调用send()进行发送。

ask()方法和send()方法在[Spark源码阅读6：RpcEnv](./master/docs/rpcenv.md)中已经遇见过，先分析ask()方法，它的作用是"异步发送一条消息，并在指定
的超时时间内等待RPC端点的回复"。从代码实现上看，ask()方法的执行分为两种情况：
  * 如果远端地址与当前NettyRpcEnv的地址相同，那么说明处理该消息的RPC端点就在本地，此时新建Promise对象，将其Future设置为回调方法(即onSuccess()方法
  和onFailure()方法)，并调用调度器的postLocalMessage()方法将消息发送给本地的RPC端点。

  * 如果远端地址与当前NettyRpcEnv的地址不同，则说明处理该消息的RPC端点位于其它节点，此时先将消息序列化，并将其与onSuccess()、onFailure()方法逻辑一起
  封装到RpcOutboxMessage中并投递出去。

最后用timeoutScheduler设置一个定时线程(这个timeoutScheduler就是上面NettyRpcEnv中的那个)，用来控制超时，如果超时会抛出TimeoutException，如果没有
超时，就调用cancel()方法取消计时。

send()方法的作用是"发送一条单向的异步消息，使用'发送即忘'语义，无需回复"，实现逻辑与ask()方法大致相同，也分为远端地址与当前NettyRpcEnv的地址相同和不同
两种情况，细节上稍有区别，不再赘述。