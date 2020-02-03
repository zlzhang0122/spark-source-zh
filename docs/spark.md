### spark start

Storm、Spark Streaming(Structured Streaming)、Flink是实时流式计算领域的三大框架，由于工作原因，主要接触了其中的两种，也就是Storm和Flink，
并对这两者的底层源码进行过系统的研究和开发，但是SS却一直只停留在上层的开发层面，一直未能有时间对其底层代码进行深入的研究，其中的原因颇多，恰好最近
换工作后可以抽出来的工作之余的时间变得富余了一些，于是便来拜读顺道记录一下。

由于SS依赖于底层的Spark Core的框架，所以就先从Spark Core的源码开始阅读，好了，那就开始吧，毕竟时间有限，必须争分夺秒。

Spark Core的运行架构图如下图所示：
![Spark运行架构图](../image/spark-runtime.png "Spark运行架构图")

其中，Resource Scheduler可以是YARN、Mesos、Kubernetes(K8s)、Spark Standalone(自带的资源管理器)，主要负责资源的分配和监控，Driver
负责作业逻辑的调度和任务的监控。根据部署模式的不同，启动和运行的物理位置也有所不同。在Client模式下，Driver模块运行在Spark-Submit进程中。
Cluster模式下，Driver的启动过程与Executor类似，运行在资源调度器分配的资源容器内。

此外，还有一些概念也值得注意：
  * RDD：弹性分布式数据集，是Spark中最重要的概念之一，它是一个可以并行的、可容错的数据集合;

  * Application：简单来说，用户每次提交的所有代码就是一个Application;

  * Job：一个Application可以分为多个Job，每次触发一个final RDD的计算就是一个Job;

  * Stage：一个Job可以分为多个Stage，它根据Job中RDD的依赖关系来分，每遇到一个宽依赖就会划分成一个新Stage;

  * Task：是最小最基本的计算单位，一般是数据的一个分块是一个Task，大小为128M;