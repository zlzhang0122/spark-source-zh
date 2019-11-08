### Stage的划分

在[Spark源码阅读3：Spark任务提交](./jobsubmit.md)中已经讲过了Spark Job的提交，在其中提到，当rdd触发action操作后，都会
调用SparkContext的runJob方法，并调用DAGScheduler.handleJobSubmitted方法完成整个job的提交。DAGScheduler会根据RDD的lineage进行Stage
的划分，生成TaskSet，并由TaskScheduler向集群申请资源，最终在Worker节点的Executor进程中执行Task。

先看一下如何进行Stage的划分，如下图所示是对应Spark应用程序代码生成的Stage。它根据RDD的依赖关系进行划分，在遇到宽窄依赖时将两个RDD划分为不同
的Stage。

![Stage划分](../image/stage.png "Stage划分")