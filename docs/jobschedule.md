### Task调度

在[Spark源码阅读5：Stage的提交](./stagesubmit.md)已经说到，最终stage被封装成TaskSet，并使用taskScheduler.submitTasks进行提交。
TaskScheduler负责低层次任务的调度，每个TaskScheduler为一个特定的SparkContext调度tasks。这些调度器获取到由DAGScheduler为每个stage
提交至他们的一组Tasks，并负责将这些tasks发送到集群，以执行它们，在它们失败时重试，并减轻掉队情况。
