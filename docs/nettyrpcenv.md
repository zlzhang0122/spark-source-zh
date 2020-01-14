### NettyRpcEnv

NettyRpcEnv作为RPC环境，除了拥有接收消息的功能外，也应该能发送消息。那我们就先来看一看NettyRpcEnv中与消息发送相关的成员：
  * clientFactory、fileDownloadFactory：都是TransportClientFactory类型，通过transportContext.createClientFactory()方法创建，这个工厂类
  在NettyRpcEnv中用于生产TransportClient。clientFactory用于处理一般性的请求发送和应答，而fileDownloadFactory专用于下载文件，按需创建，不会立即
  被初始化。
