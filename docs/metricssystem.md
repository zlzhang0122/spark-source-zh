### MetricsSystem

MetricsSystem负责收集、存储和输出度量指标，对于大数据处理引擎而言，系统监控与功能的实现一样重要。比如Storm、Flink都有相应的metric系统来进行
系统和任务运行指标的收集、存储和上报，Spark也同样不例外，而了解Spark MetricsSystem的相关细节对于研究Spark同样也很有意义。

MetricsSystem有三个主构造方法参数，分别是：
  * instance：表示该度量系统对应的实例名，可取值有"master"、"worker"、"executor"、"driver"、"applications"或者"*"等

  * conf：SparkConf配置项

  * securityMgr：安全管理器SecurityManager实例

MetricsSystem中的属性成员有：
  * metricsConfig：度量系统的配置，是MetricsConfig的实例，用于设置和加载度量配置

  * sinks：度量目的地的缓存数组，度量目的地就是输出及展现监控指标的组件，继承于Sink特征

  * sources：度量来源的缓存数组，度量来源是产生及收集监控指标的组件，继承于Source特征

  * registry：度量注册中心，com.codahale.metrics.MetricRegistry的实例，source和sink通过它来注册到度量仓库，度量仓库是Codahale提供的度量组件，
  Spark以此为基础构建自己的度量系统

  * running：表示当前MetricsSystem是否正在运行

  * metricsServlet：可以看作是一个特殊的sink，提供给Web UI使用

