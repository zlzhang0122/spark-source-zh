### ExecutorAllocationManager

ExecutorAllocationManager是Executor分配管理器，它可以通过与集群管理器联系并根据当前的负载动态增加或删除Executor，属于一个比较智能的机制。

先来分析一下它的初始化的具体流程：
  * 判断是否启用Executor动态分配，如果spark.dynamicAllocation.enabled配置项为true，并且满足以下两个条件之一：spark.dynamicAllocation.testing
  配置项为true，或者当前不是本地模式，就启用Executor动态分配;

  * 判断SchedulerBackend的实现类是否继承了ExecutorAllocationClient特征，如果使得，就用SchedulerBackend、ListenerBus、SparkConf和BlockManagerMaster
  的实例构造出ExecutorAllocationManager;

  * 调用ExecutorAllocationManager.start()方法启动;

