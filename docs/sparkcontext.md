### SparkContext构建

对Spark有所了解的同学都知道，SparkContext绝对是Spark编程非常重要且经常接触的一个东西。它是开发Spark应用的入口，负责和整个集群的交互，包括
申请集群资源、创建RDD、累加器和广播变量等。理解Spark的架构就必须从这个入口开始。下图是官网的一张Spark架构图：

![Spark架构图](../image/spark.png "Spark架构图")

对于该图官网有几点说明：<br/>
（1）不同的Spark应用程序对应着不同的Executor，这些Executor在整个应用程序执行期间都存在，并且可以以多线程的方式执行Task。(Hmmmm....回忆
一下，Storm中也有Executor，但Storm中的Executor是一个单独的线程，默认里面有一个Task，如果设置多个，同一个Executor中的Task必须是同类型的，
要么是Spout，要么是bolt，并且多个Task以串形方式执行。Flink中呢没有Executor，与之对应的个人觉得应该是Slot，Slot中以多线程的方式运行Task，
每个Task是一个线程。)Spark这样左的好处是，各个Spark应用程序的执行是相互隔离的，除了Spark应用程序向外部存储系统写数据进行数据交互外，无法进行
其他形式的数据共享。


其中，DriverProgram就是用户提交的程序，里面就定义有SparkContext的的实例。SparkContext默认的构造函数接受的是org.apache.spark.SparkConf，
通过这个参数我们可以自定义提交的参数，这个参数会覆盖系统提供的默认配置。