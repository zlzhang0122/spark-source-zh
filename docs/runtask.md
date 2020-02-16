### Task执行

在[Spark源码阅读32：Task调度](./taskschedule.md)已经提到，在调用resourceOffers()为Tasks分配资源后，会调用launchTasks()启动Tasks。在launchTasks()
方法中，循环Tasks集合，对每一个Task进行序列化。如果序列化的Task的大小超过了spark.rpc.message.maxSize配置项的配置，默认是128MB，则退出当次所有正在运行中
的Tasks，表示任务执行失败。否则，就将对应的executor的空闲CPU核数减掉需要占用的数目，同时向对应的executor发送LaunchTask消息，消息中包含了需要执行的Task的
信息。

在Standalone模式下，该消息的接收者是CoarseGrainedExecutorBackend，它继承了ThreadSafeRpcEndpoint trait，并对其中的receive方法进行了实现。直接来看
对LaunchTask消息的处理，首先对Task信息进行反序列化，然后调用了executor.launchTask()方法，在这个方法中，它创建了一个TaskRunner内部类，该内部类是一个线程，
线程中的run()方法用于完成Task的执行。该方法很长，但是实际上逻辑非常简单，就是执行前向Driver端发状态更新，然后将获取到的Task进行反序列化，获取需要的配置和
依赖，然后调用task.run()方法去执行，执行完成后，通知Driver端进行状态更新，并更新metrics信息，对执行异常进行处理，最后将Task从运行任务列表中删除。

直接来看task.run()方法，该方法创建Task运行上下文，然后将Task的上下文对象通过runTask()方法传入，runTask()方法是真正执行任务的方法，前面提到Spark中有两种Task，
分别是ShuffleMapTask和ResultTask。此处runTask()方法根据传入的Task的不同，会调用对应runTask()方法的实现。具体的执行方法是，先获取DAGScheduler.submitMissingTasks()
中序列化得到的Task广播变量，对其进行进行反序列化，得到RDD和执行函数，然后通过rdd.iterator()方法，对其partition进行遍历计算得到结果，不同的是ShuffleMapTask需要shuffle write，
以供child stage读取shuffle的结果。

在上面的TaskRunner.run()方法中多次提到向Driver端发状态更新，实际上进行状态更新时调用的是CoarseGrainedExecutorBackend.statusUpdate()方法，向Driver端发送StatusUpdate消息，
DriverEndpoint.receive()方法接收并处理发送过来的StatusUpdate消息，调用TaskSchedulerImpl.statusUpdate()方法更新，在这个方法中。它判断如果是执行成功调用
taskResultGetter.enqueueSuccessfulTask()方法从远端Worker节点的BlockManager当中获取计算结果，然后通过DAGScheduler.handleTaskCompletion()方法对任务结束进行处理。
如果执行结果是FAILED、KILLED、LOST，则调用taskResultGetter.enqueueFailedTask()方法处理任务失败的情况，并最终调用DAGScheduler.handleTaskCompletion()处理任务结束的情况。

至此，任务的执行就已经分析完毕，我们对基于RDD的任务调度系统应该也有了一个大致的了解。
终于，完成了对Spark Core的核心部分代码的阅读工作。因为武汉新型冠状病毒的影响，日常不能出去使得阅读时间变得更多，完成时间也比预计的要早不少。。。。。。