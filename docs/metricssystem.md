### MetricsSystem

MetricsSystem负责收集、存储和输出度量指标，对于大数据处理引擎而言，系统监控与功能的实现一样重要。比如Storm、Flink都有相应的metric系统来进行
系统和任务运行指标的收集、存储和上报，Spark也同样不例外，而了解Spark MetricsSystem的相关细节对于研究Spark同样有很重要的意义。

MetricsSystem的实例化可以通过调用MetricsSystem.createMetricsSystem()方法，通过这个方法内部的实现可以看到，MetricsSystem有三个主构造方法
参数，分别是：
  * instance：表示该度量系统对应的实例名，可取值有"master"、"worker"、"executor"、"driver"、"applications"或者"*"等;

  * conf：SparkConf配置项;

  * securityMgr：安全管理器SecurityManager实例;

MetricsSystem中的属性成员有：
  * metricsConfig：度量系统的配置，是MetricsConfig的实例，用于设置和加载度量配置;

  * sinks：度量目的地的缓存数组，度量目的地就是输出及展现监控指标的组件，继承于Sink特征;

  * sources：度量来源的缓存数组，度量来源是产生及收集监控指标的组件，继承于Source特征;

  * registry：度量注册中心，com.codahale.metrics.MetricRegistry的实例，source和sink通过它来注册到度量仓库，度量仓库是Codahale提供的度量组件，
  Spark以此为基础构建自己的度量系统;

  * running：表示当前MetricsSystem是否正在运行;

  * metricsServlet：可以看作是一个特殊的sink，提供给Web UI使用;

MetricsSystem提供了registerSource()方法来注册单个度量来源，这个方法首先将度量来源加入到sources缓存数组中，然后调用buildRegistryName()方法来构
造Source的注册名称，最后调用MetricRegistry.register()方法将其注册到Metrics仓库。Source的注册名称取决于Metrics的命名空间和由spark.executor.id
配置项控制的Executor ID，而命名空间又由spark.metrics.namespace配置项控制，它有一个默认值，默认值由spark.app.id配置项控制，也就是Application ID。
注册名称还有一个默认值，默认值由MetricRegistry.name()方法来生成。

在MetricsSystem启动时，会根据MetricsConfig配置调用registerSources()方法来注册所有的source，具体来讲，它会先通过getInstance()方法来获取实例名
下的所有配置，然后根据正则表达式^source\\.(.+)\\.(.+)匹配得出该实例的配置中所有与source有关的配置，返回值类型是mutable.HashMap[String, Properties]，
这个类型是scala中的HashMap的实现。最后，根据配置中的class属性，Utils.classForName()通过反射构造出source实现类的对象实例，调用registerSource()
方法将source注册到Metrics仓库。

与source的注册不同，MetricsSystem并没有提供registerSink来注册单个度量目的地的方法，而只是提供了registerSinks()方法在初始化时批量注册sink。这个方法
前面的部分跟registerSources()完全一样，除了正则表达式换成了^sink\\.(.+)\\.(.+)以匹配该实例所有与sink相关的配置参数。然后也是利用Utils.classForName()
方法反射构造sink实现类的实例，只是此处调用的是有三个参数的有参构造函数，而source时调用的是无参构造函数。在获得sink的实现类的对象实例后，如果度量实例名称是
servlet，说明是提供给web ui使用的sink，则将其赋值给MetricsServlet属性，否则放入sinks缓存数组。在MetricsSystem启动方法的最后，会分别调用每一个sink的
start()方法启动Sink。

上面已经提到，metricsConfig用于设置和加载度量配置，metricsConfig.initialize()方法用于其的初始化，在MetricsSystem的构造方法中进行了调用，其初始化流程是：
首先调用setDefaultProperties()方法加载默认的配置(主要是避免没有进行过配置的情况)，然后调用loadPropertiesFromFile()方法从文件中加载配置，文件路径由
spark.metrics.conf配置项指定。然后在SparkConf中查找"spark.metrics.conf."字符串前缀的配置项，将键的后缀及值都放入到properties中。最后调用subProperties()
方法通过正则匹配分拆出各个实例的配置，保存到perInstanceSubProperties属性中(数据类型是mutable.HashMap[String, Properties])。

在上面的setDefaultProperties()方法中设置的默认属性共有4个，分别是*.sink.servlet.class、*.sink.servlet.path、master.sink.servlet.path、
applications.sink.servlet.path，而loadPropertiesFromFile()方法首先从spark.metrics.conf配置的文件路径中加载配置文件，如果没有提供，则从类路径的
metrics.properties文件中读取。

在上面调用subProperties()方法通过正则匹配分拆出各个实例的配置时，度量配置的格式是：[instance].[sink|source].[name].[options]=value，比如
master.sink.servlet.path=/metrics/master/json。拆分的结果就是原key的instance部分作为HashMap的key，原key的剩余部分作为Properties的key，原value部分
作为Properties的value，最终结果就是按照instance名称对度量配置进行分组。

经过上面的一番分析，已经可以明确，Spark的Metrics系统是由Instance、Source、Metrics、Sink四个部分组成，它们之间的关系如下图：
// todo (假装我是图，真的以后补)

Source是一个非常简单的trait，其中定义了两个方法，sourceName获取度量来源的名称，metricRegistry获取其对应的注册中心。它有多种具体的实现，其中executor的
度量来源实现ExecutorSource是其中比较常见和重要的，我们以它为重点来简单描述Source的具体实现：ExecutorSource向注册中心注册了很多指标，包括与threadpool
相关的Guage、与filesystem相关的Guage，Guage是Metrics体系内估计metrics值的工具。还有着大量的计数器用于统计GC、shuffle、serializable等方面的计数值。

Sink也是一个比较简单的trait，其中定义了3个方法：start()/stop()方法分别用于启动和停止Sink，report()方法用于输出具体的metrics的值，其有七个具体的实现。
具体来说，ConsoleSink输出到控制台，CsvSink输出到CSV文件。Slf4jSink对Codahale Metrics中的Slf4jReporter类进行简单封装，然后Slf4jReporter启动之后，
就会按照pollPeriod和pollUnit指定的时间周期性的轮询METRics值并输出到符合SLF4J规范的日志等。此外，JmxSink可以通过将metrics数据输出到JMX中，从而通过
JVM可视化工具(如VisualVM)进行查看。而MetricsServlet在上面提到过，可以利用Spark UI内置的Jetty服务输出metrics数据到浏览器。
// todo(假装我是图，真的以后补)