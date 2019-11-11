### Stage的提交

在[Spark源码阅读3：Spark任务提交](./jobsubmit.md)里已经介绍到了任务提交时会调用handleJobSubmitted进行任务的提交，接着继续往下追，其后
的调用流程是：
org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted
org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted.submitStage
org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted.submitMissingTasks
org.apache.spark.scheduler.TaskScheduler.submitTasks

在handleJobSubmitted方法中，在创建好finalStage并做好相关的设置后，最后会调用submitStage将其进行提交，在通过submitStage提交finalStage时，
方法会递归的将finalStage依赖的父stage先提交，最后再提交finalStage。