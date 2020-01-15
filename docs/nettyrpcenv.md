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

  * outboxes：在[Spark源码阅读5：RpcEnv](../master/docs/rpcenv.md)前面我们在研究dispatcher时看到了
