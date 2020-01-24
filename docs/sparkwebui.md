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

WebUI是Spark中所有可以在浏览器中展示的内容的顶级组件，SparkUI类也继承于它。其中有6个属性成员：
  * tabs：持有Web UI Tab的缓存;

  * handlers：持有Jetty ServletContextHandler的缓存;

  * pageToHandlers：保存WebUI Page(Web UI Tab的下一级组件)及其对应的ServletContextHandler的对应关系;

  * serverInfo：Web UI对应的Jetty服务器的信息;

  * publicHostName：Web UI对应的Jetty服务器名。先通过系统环境变量SPARK_PUBLIC_DNS获取，如果为空再通过spark.driver.host配置项获取;

  * className：经过Utils.getFormattedClassName()方法格式化后的当前类名。

Getter方法有4个，getTabs()和getHandlers()方法就是简单地获取对应属性的值，getBasePath()获取构造参数中定义的Web UI基路径，getSecurityManager()
则取得构造参数中传入的安全管理器。




