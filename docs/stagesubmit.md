### Stage的提交

在[Spark源码阅读30：Spark任务提交](./jobsubmit.md)里已经介绍到了任务提交时会调用handleJobSubmitted进行任务的提交，在其中提到，当rdd触发
action操作后，都会调用SparkContext的runJob方法，并调用DAGScheduler.handleJobSubmitted方法完成整个job的提交。接着继续往下追，其后的调用
流程是：

org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted()
org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted.submitStage()
org.apache.spark.scheduler.DAGScheduler.handleJobSubmitted.submitMissingTasks()
org.apache.spark.scheduler.TaskScheduler.submitTasks()

![StageSubmit调用链](../image/stagesubmit.png "StageSubmit调用链图")

其中，在handleJobSubmitted()方法中，DAGScheduler会根据RDD的lineage进行Stage的划分，生成Stage，包括最后一个Stage：ResultStage和前面的
ParentStage：ShuffleMapStage，生成TaskSet，并由TaskScheduler向集群申请资源，最终在Worker节点的Executor进程中执行Task。

先看一下如何进行Stage的划分，如下图所示是对应Spark应用程序代码生成的Stage。它根据RDD的依赖关系进行划分，在遇到宽窄依赖时将两个RDD划分为不同
的Stage(什么是宽依赖，什么是窄依赖，可以看[Spark源码阅读26：RDD](./rdd.md)进行了解)。
![Stage划分](../image/stage.png "Stage划分")

在上图中可以看到，RDD G与RDD F间的依赖是宽依赖，所以RDD F与RDD G被划分为不同的Stage，而RDD B与RDD G之间是窄依赖，因此RDD B与RDD G被
划分为同一个Stage。通过这种递归的调用方式，将所有的RDD进行划分。实际上[Spark源码阅读30：Spark任务提交](./jobsubmit.md)里也已经大致介绍过，
最后一个Stage也就是finalStage的创建是通过触发action来实现的。在递归生成Stage时具体区分宽依赖与窄依赖的办法是看依赖关系是否为ShuffleDependency，
如果是则表示是宽依赖，需要重新生成Stage，否则就可以属于同一个Stage，生成完成finalStage后就会进行Stage的提交。

具体来说，在DAGScheduler.createResultStage()方法中会调用getOrCreateParentStages()方法获取Parent Stages，获取方式是采用广度优先遍历(这个遍历
方式应该挺熟悉的，它就是一种典型的通过引入Stack来实现非递归遍历的办法，我们在写二叉树的非递归遍历方式时经常有类似的写法)的方式根据宽依赖ShuffleDependency
获取到其直接的宽依赖(非直接的宽依赖不会被获取到)，然后为每一个获得的宽依赖生成ShuffleMapStage，生成方式是如果该宽依赖对应的Stage已经存在则不需要创建，
直接原样返回。否则就表示需要创建，在创建之前会先判断其依赖的祖先Stage是否存在，不存在则需要先进行创建，创建完祖先Stage完成后再调用createShuffleMapStage()
创建当前ShuffleDependency对应的Stage，最后创建ResultStage，其id是通过AtomicInteger类型的getAndIncrement()获得，能够保证原子性，这个ResultStage就是
整个Job的finalStage，将这个finalStage加入到数据结构stageIdToStage中，并更新数据结构jobIdToStageIds，最后将这个ResultStage返回。至此已根据ShuffleDependency
对Stage完成了划分。

在创建好finalStage并做好相关的设置后，最后会调用submitStage()将其进行提交。在通过submitStage()提交finalStage时，方法会递归的将finalStage依赖的父stage先提交，
最后再提交finalStage，提交时调用submitMissingTasks方法。代码的具体逻辑比较简单：
  * 先根据stage获取到jobId，如果jobId没有定义，说明该Stage不属于运行中的Job，则调用abortStage()方法放弃该stage。如果jobId已经定义，则需要判断该stage是否属于
  waitingStages、runningStages、failedStages中的任意一个，若是则忽略不处理。waitingStages表示等待处理的stages，由于Spark采取由前往后的顺序处理stage的提交，即先处理
  parent stage，然后再处理child stage，所以在waitingStages中的stage由于其parent stage尚未处理，所以必须先等待parent stage的处理。runningStages表示正在运行的stage，
  正在运行表示已经被提交，无需重复提交。failedStages表示失败的stage，已经失败的stage再次提交仍然会失败，所以也无需提交。如果stage不存在于上述三个数据结构中，就可以继续执行
  后续的提交流程;

  * 然后，调用getMissingParentStages()方法获取stage还没有提交的parent，即missing。若missing为空，说明该stage要么没有parent stage，要么其parent stages都已经被提交，
  此时该stage就可以被提交，调用submitMissingTasks()进行提交；若missing不为空，说明该stage还存在尚未被提交的parent stages，就需要遍历missing，循环提交每个stage，并将
  该stage添加到waitingStages中，等待其parent stages都被提交后再被提交。在getMissingParentStages()方法中，定义了三个数据结构和一个visit()方法。三个数据结构分别是：
  (1) missing：HashSet[Stage]类型，存储尚未提交的parent stages，用于最后结果的返回;
  (2) visited：HashSet[RDD[_]]类型，已被处理的RDD集合，位于其中的RDD不会被重复处理;
  (3) waitingForVisit：ListBuffer[RDD[_]]类型，等待被处理的RDD集合;
  visit()方法的处理逻辑也比较简单：通过RDD是否在visited中判断RDD是否已处理，若未被处理，添加到visited中，然后循环rdd的dependencies，如果是宽依赖ShuffleDependency，
  调用getOrCreateShuffleMapStage()，获取ShuffleMapStage（此次调用则是直接取出已生成的stage，因为划分阶段已将stage全部生成），判断该stage的isAvailable标志位，若为false，
  则说明该stage未被提交过，加入到missing集合，如果是窄依赖NarrowDependency，直接将RDD压入waitingForVisit栈，等待后续处理，因为窄依赖的RDD同属于同一个stage，加入
  waitingForVisit只是为了后续继续沿着DAG图继续往上处理。那么，整个missing的获取就一目了然，将final stage即ResultStage的RDD压入到waitingForVisit顶部，循环处理即可得
  到missing;

在submitMissingTasks方法中，在查找missing(也就是未提交的stages)的分区前，先进行中间状态的清除工作，这个操作能够保证，对于部分完成的中间状态，findMissingPartitions()
方法每次都能返回所有的分区。具体办法是：判断stage是否是ShuffleMapStage，在DAG Stage中，除了最后的Stage外，其余的全部都是ShuffleMapStage。如果是ShuffleMapStage，并且
还没有被提交，则先执行中间Stage的清理工作。然后调用findMissingPartitions()方法确定该stage需要计算的分区ID索引，保存到partitionsToCompute。将stage加入到runningStages
中去，标记stage正在运行。当一个stage启动时，需要先调用outputCommitCoordinator的stageStart进行初始化。然后创建一个Map：taskIdToLocations，存储的是id->Seq[TaskLocation]
的映射关系，并对stage中指定RDD的每个分区获取位置信息，映射成id->Seq[TaskLocation]的关系，注意此处的getPreferredLocs()方法，很重要，它用于保证数据的本地性，是task获取最佳
执行位置的算法。标记新的stage attempt关系，并发送一个SparkListenerStageSubmitted事件。对stage进行序列化并广播，此时有两种情况，如果是ShuffleMapStage，则序列化rdd和
shuffleDep；如果是ResultStage，序列化rdd和func。接下来是一个比较重要的步骤，针对stage的每个分区构造task，形成tasks，因此stage的每个分区都会对应一个task，其中ShuffleMapStage
生成ShuffleMapTasks，ResultStage生成ResultTasks。如果tasks不为空，则利用taskScheduler.submitTasks()提交task，否则标记stage已完成。


