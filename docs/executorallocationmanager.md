### ExecutorAllocationManager

ExecutorAllocationManager是Executor分配管理器，它可以通过与集群管理器联系并根据当前的负载动态增加或删除Executor，属于一个比较智能的机制。

先来分析一下它的初始化的具体流程：
  * 判断是否启用Executor动态分配，如果spark.dynamicAllocation.enabled配置项为true，并且满足以下两个条件之一：spark.dynamicAllocation.testing
  配置项为true，或者当前不是本地模式，就启用Executor动态分配;

  * 判断SchedulerBackend的实现类是否继承了ExecutorAllocationClient特征，如果使得，就用SchedulerBackend、ListenerBus、SparkConf和BlockManagerMaster
  的实例构造出ExecutorAllocationManager;

  * 调用ExecutorAllocationManager.start()方法启动;

再来分析一下ExecutorAllocationManager类的成员属性：
  * minNumExecutors/maxNumExecutors：分别对应配置项spark.dynamicAllocation.minExecutors/maxExecutors，代表动态分配过程中最小和最大的Executor
  数量，默认是0和Int.MaxValue;

  * initialNumExecutors：Executor的初始数量，值由Utils.getDynamicAllocationInitialExecutors()方法来指定，其值是spark.dynamicAllocation.minExecutors、
  spark.dynamicAllocation.initialExecutors、spark.executor.instances三个参数的较大值;

  * tasksPerExecutor：每个Executor执行的Task数量，由spark.executor.cores除以spark.task.cpus得到;

  * schedulerBacklogTimeoutS：由spark.dynamicAllocation.schedulerBacklogTimeout配置项确定，表示当有Task等待调度超过该时长时，就开始动态分配资源，默认1s;

  * sustainedSchedulerBacklogTimeoutS：由spark.dynamicAllocation.sustainedSchedulerBacklogTimeout配置项指定，表示动态分配资源仍未达标时，每次再分配的时
  间间隔，默认与schedulerBacklogTimeoutS相同;

  * executorIdleTimeoutS：由spark.dynamicAllocation.executorIdleTimeout配置项指定，表示Executor处于空闲状态(没有执行Task)的超时，超时后会移除Executor，
  默认值为60s;

  * cachedExecutorIdleTimeoutS：由spark.dynamicAllocation.cachedExecutorIdleTimeout配置项指定，表示持有缓存块的Executor的空闲超时。由于缓存不能随意被
  清理，因此其默认值为Integer.MAX_VALUE;

  * numExecutorsToAdd：下次动态分配要增加的Executor数量;

  * numExecutorsTarget：在当前时刻的Executor目标数量，这个计数主要是为了在Executor突然大量丢失的异常情况下，能够快速申请到需要的数目;

  * executorsPendingToRemove：即将被移除但还没被杀掉的Executor ID缓存;

  * executorIds：所有目前已知的Executor ID缓存;

  * addTime：本次触发Executor添加的时间戳;

  * removeTimes：Executor将要被删除时的ID与时间戳的映射;

  * listener：ExecutorAllocationListener类型的监听器，用于监听与Executor相关的事件，包括Stage和Task提交与完成，Executor添加与删除等;

  * executor：单线程的调度线程池，用来执行周期性检查并动态分配Executor的任务;

  * localityAwareTasks：所有当前活跃的Stage中，具有本地性偏好（就是数据尽量位于本地节点）的Task数量;

  * hostToLocalTaskCount：每个物理节点上运行的Task数目的近似值;

最后来分析一下ExecutorAllocationManager类的提供的成员方法：
  * start()：先将ExecutorAllocationListener注册到LiveListenerBus中。然后会创建执行schedule()方法的任务，并用调度线程池executor以默认100ms的间隔定期执行。最
  后，调用ExecutorAllocationClient的requestTotalExecutors()方法，请求分配Executor;

  * schedule()：主要做了两件事，一是调用updateAndSyncNumExecutorsTarget()方法重新计算并同步当前所需的Executor的数量，二是找出那些已经过期的Executor并调用
  removeExecutors()方法删除它们;

  * updateAndSyncNumExecutorsTarget()：调用maxNumExecutorsNeeded()方法计算出当前所需的最大Executor数量maxNeeded。其计算方法是：从监听器取得等待中的Task
  计数与运行中的Task计数，将两者相加并减1，最后除以每个Executor上运行Task数的估计值。如果ExecutorAllocationManager仍然在初始化，就直接返回0(这个返回值是
  Executor数量的变化量，而不是总数)；否则，检查maxNeeded与前述numExecutorsTarget值的大小关系，如果目标Executor数量超过了最大需求数，就将numExecutorsTarget
  设置为maxNeeded与minNumExecutors的较大值，然后调用ExecutorAllocationClient.requestTotalExecutors()方法。此时会通知集群管理器取消未执行的Executor，并且
  不再添加新的Executor，返回减少的Executor数量；如果目标Executor数量小于最大需求数，并且当前的时间戳比上一次添加Executor的时间戳要新，就调用addExecutors()方法，
  此时会通知集群管理器新添加Executor，更新addTime记录的时间戳，返回增加的Executor数量;

  * onExecutorIdle()：首先确定removeTimes和executorsPendingToRemove缓存中都不存在当前的Executor ID，然后判断该Executor是否缓存了块。如果有缓存块，就将其超时
  时间设为Long.MaxValue，否则就按正常的空闲超时来处理。最后将这个Executor的ID与其计划被删除的时间戳存入removeTimes映射;

  * removeExecutors()：首先计算剩余的Executor数目，然后遍历要删除的Executor ID列表，判断删除之后剩余的Executor数是否小于最小允许的Executor数量与目标Executor
  数量，如果是的话，该Executor就不能删除。反之，如果根据canBeKilled()方法判断出executorIds缓存中存在该Executor，并且尚未进入executorsPendingToRemove，就将其
  标记为可删除。然后调用ExecutorAllocationClient.killExecutor()方法，真正地杀掉Executor。再调用requestTotalExecutors()方法，重新申请新的Executor数目。如果
  要删除的Executor列表中有最终未被杀掉的，就将它们再次加入executorsPendingToRemove缓存中，等待删除。最后，监听器会调用Executor减少后的回调方法onExecutorRemoved()，
  该方法主要是清理各个缓存，逻辑比较简单;

  * addExecutors()：这个方法中需要注意的是，当增加Executor时，每次申请的新Executor数目是指数级别增长的，之所以采用这种策略是因为从经验上来说，多数App在启动时只需要少量
  的Executor就可以满足计算需求，但一旦资源紧张时，用指数增长可以使申请到满足需求的资源的次数降低;