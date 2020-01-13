### Dispatcher

在[Spark源码阅读5：RpcEnv](../master/docs/rpcenv.md)中已经介绍过，Dispatcher是NettyRpcEnv中所包含的组件，用于将消息路由到正确的RPC端点。
它里面的属性不是很多，但是比较重要，我们一个一个的看一看：
  * endpoints/endpointRefs：这是两个ConcurrentHashMap，分别用来维护RPC端点名称与端点数据EndpointData的映射，以及RPC端点与其引用的映射。

  * receivers：存储RPC端点数据的阻塞队列，LinkedBlockingQueue类型，当RPC端点收到要处理的消息时，会将其放入这个队列，空闲的RPC端点无消息可放。

  * threadpool：用来分发消息的固定大小的守护线程池，线程数由spark.rpc.netty.dispatcher.numThreads确定，默认值是2和可用CPU核数两者中的最大值，
  线程池内的线程都是MessageLoop类型的。(最新版3.0中已被替换)

  * EndpointData：是Dispatcher中的私有内部类，实现很简单，接收三个参数：RPC端点名称、RPC端点实例、端点实例的引用，然后创建一个Inbox的实例，Inbox
  是一个"收件箱"，它存储着对应RPC端点所收到且需要处理的消息，这些消息都继承自InboxMessage特征，且向Inbox发送消息是线程安全的。(最新版3.0中已被替换)

前面已经提到Dispatcher的线程池执行的都是MessageLoop，这是一个内部类，本质上是一个线程，不断循环的处理消息。其从reveivers阻塞队列中获取EndpointData，
判断如果是PoisonPill(一种特殊的端点信息，标志着循环的结束)，则在线程结束前会将其重新放回队列中，以结束其它活着的线程；如果不是PoisonPill，则调用
Inbox.process()方法，对该RPC端点收件箱中的消息进行处理。由于take会阻塞的获取消息，所以如果receivers队列为空会阻塞直到有新的EndpointData进入。

