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

Web UI提供了一些attach/detach方法，这些方法都是成对出现，一共3对：attachTab()/detachTab()用于注册和移除Web UI Tab；attachPage()/detachPage()
用于注册和移除Web UI Page；attachHandler()/detachHandler()用于注册和移除ServletContextHandler。我们来着重分析下attachPage()方法，它的流程是
调用Jetty工具类JettyUtils的createServletHandler()方法，为Web UI Page的渲染方法render()和renderJson()方法创建ServletContextHandler，也就是
一个Web UI Page需要对应两个处理器。然后，调用上述attachHandler()方法向Jetty注册处理器，并将映射关系写入到handlers结构。

Spark Web UI实际上是三层的树形结构，根节点为Web UI，中层节点为Web UI Tab，叶子节点为Web UI Page，UI界面的展示主要靠Web UI Tab与Web UI Page来
实现。在Spark UI界面中，一个Tab可以包含一个或多个Page，并且Tab可选。由于一个Tab可以包含多个Page，因此pages数组用于缓存Tab下所有的Page，attachPage()
方法用于将Tab的路径前缀与Page的路径前缀拼合起来，并加入到pages数组中。Web UI Tab与Web UI Page各有很多实现，分别对应一个Tab或一个Page。

Spark UI Tab是对Web UI Tab的简单封装，加上Application名称和Spark版本的属性。Environment Tab类只有构造方法，它会调用上面预先定义好的attachPage()方法，
将Environment Page加入。Environment Page的render()方法用来渲染页面内容，流程如下：
  * 从AppStatusStore中获取所有环境信息;

  * 调用UIUtils.listingTable()方法，将对应的表头与添加了HTML标签的行封装成表格;

  * 将表格排列好，调用UIUtils.headerSparkPage()方法，按定义好的页面布局展示在浏览器上。
