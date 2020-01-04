### SparkEnv

SparkEnv也是SparkContext中的重要组件，Driver和Executor的正常运行都离不开SparkEnv提供的环境，当SparkEnv初始化完成后，与Spark存储、计算、
监控相关的底层功能才真正准备好，可见其重要性。

在前面的源码研究过程中，我们已经知道Driver执行环境是通过调用SparkEnv.createDriverEnv()方法来创建的，这个方法在SparkEnv的伴生对象中。当然，
同理就会有createExecutorEnv()方法，它与createDriverEnv()方法类似，都是调用伴生对象内的create()方法来创建SparkEnv的，只是传递的方法略有
不同，其中executorId是Executor的唯一标识，如果是Driver的话，值是字符串的"driver"，bindAddress/advertiseAddress分别是监听socket绑定的
地址和RPC端点的地址，isLocal标识是否为本地模式启动，numUsableCores表示分配给Driver或Executor的CPU核数，ioEncryptionKey是I/O加密的密钥，
当spark.io.encryption.enable项启用时才会生效。



