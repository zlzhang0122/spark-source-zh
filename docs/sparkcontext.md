### SparkContext构建

对Spark有所了解的同学都知道，SparkContext绝对是Spark编程非常重要且经常接触的一个东西。它是开发Spark应用的入口，负责和整个集群的交互，包括
创建RDD、累加器和广播变量等。理解Spark的架构就必须从这个入口开始。下图是官网的一张Spark架构图：
![Spark架构图](../image/spark.png "Spark架构图")