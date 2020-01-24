### Spark Web UI

上一节已经讲过了MetricsSystem，今天可以就分析一下与之有密切关联关系的Spark Web UI。Spark Web UI主要依赖于Servlet容器Jetty实现，Jetty就不说了，
这里主要讲与SparkUI相关的。

SparkUI的创建是在SparkContext中，代码为SparkUI.create(Some(this), _statusStore, _conf, _env.securityManager, appName, "", startTime)，
如果spark.ui.enabled为true，表示启用了SparkUI，则使用上述代码创建。其中，_statusStore是已经初始化的AppStatusStore,它是经过包装的KVStore和
AppStatusListener，KVStore用于存储监控数据，而AppStatusListener注册到事件总线中的appStatus队列中。_env.securityManager是SparkEnv中初始化
的安全管理器。

SparkUI类中拥有3个属性成员：
  * killEnabled：由spark.ui.killEnabled配置项控制，若为true，则会在UI界面中展示强行杀掉Spark Job的开关;

  * appId：表示当前的Application ID;

  * streamingJobProgressListener：用于Spark Streaming作业进度的监听器;

在SparkUI的initialize()方法中，先创建了5个Tab，并调用attachTab()方法将这5个Tab注册到WebUI，这里所谓的Tab就是SparkUI中的标签页。接着，调用
createStaticHandler()方法创建静态资源的ServletContextHandler，又调用createRedirectHandler()创建一些重定向的ServletContextHandler。
(ServletContextHandler是Jetty中一个功能完善的处理器，负责接收并处理HTTP请求，再投递给Servlet。)最后，对每一个handler都调用attachHandler()
方法注册到WebUI。




