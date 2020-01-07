### Dispatcher

在[Spark源码阅读5：RpcEnv](../master/docs/rpcenv.md)中已经介绍过，Dispatcher是NettyRpcEnv中所包含的组件，用于将消息路由到正确的RPC端点。
它里面的属性不是很多，但是比较重要，我们一个一个的看一看：
  * endpoints/endpointRefs：这是两个ConcurrentHashMap，分别用来维护RPC端点名称与端点数据EndpointData的映射，以及RPC端点与其引用的映射。

  * receivers：存储RPC端点数据的阻塞队列，LinkedBlockingQueue类型，当RPC端点收到要处理的消息时，会将其放入这个队列，空闲的RPC端点无消息可放。

  * threadpool：用来分发消息的固定大小的守护线程池，线程数由spark.rpc.netty.dispatcher.numThreads确定，默认值是2和可用CPU核数两者中的最大值，
  线程池内的线程都是MessageLoop类型的。

  * EndpointData：是Dispatcher中的私有内部类，实现很简单，接收三个参数：RPC端点名称、RPC端点实例、端点实例的引用，然后创建一个Inbox的实例，Inbox
  是一个"收件箱"，它存储着对应RPC端点所收到且需要处理的消息，这些消息都继承自InboxMessage特征，且向Inbox发送消息是线程安全的。