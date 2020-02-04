### Checkpoint

突然之间，原本就已经很肆虐的疫情变得更加疯狂，而我却还没来得及买N95的口罩。往年过年还需要走亲访友，今年却是哪里也不能去。不过对于我这种不太喜欢走亲戚的人倒也乐得清闲，
还多了许多时间来分析源码，不过还是希望疫情赶快过去。武汉，加油！！！

今天来研究Spark及SS能够长时间运行非常重要的基石--Checkpoint，作为一款大数据计算框架，运行在普通的计算机上，出现异常是难免的事，既然无法避免出错，那么异常后的容错
就必不可少。检查点(Checkpoint)是Spark Core计算过程中的容错机制，它通过将RDD的数据与状态持久化，当计算过程出错时就可以从之前的状态直接恢复，而不必重新从头开始计算，
从而大大地提高了效率与可靠性。

在RDD类中，提供了一个对内的方法doCheckpoint()方法，这个方法在调度模块中提交Job时调用，并能够递归的对父RDD做Checkpoint。还提供了两个对外的方法checkpoint()
和localCheckpoint()可以对RDD做Checkpoint，它们最终都是将RDD的checkpointData属性赋值，并分别对应RDDCheckpointData的两种实现：ReliableRDDCheckpointData
和LocalRDDCheckpointData。

ReliableRDDCheckpointData和LocalRDDCheckpointData这两个类的区别从名称上就能看出来。ReliableRDDCheckpointData将CheckpointData存储在可靠的外部存储(HDFS)
文件中，恢复时从外部文件读取数据。LocalRDDCheckpointData则是将CheckpointData存储在Executor节点本地，默认存储级别是StorageLevel.MEMORY_AND_DISK，也就是同时
保存在内存和磁盘上。显然，ReliableRDDCheckpointData相比于LocalRDDCheckpointData更加安全可靠，Executor的宕机不会影响数据的恢复，但是它牺牲了速度，因此对于RDD
血缘关系过长的场景并不是很适用。

这里着重研究一下ReliableRDDCheckpointData，但是需要注意的是，必须要先设置Checkpoint的目录(通过调用SparkContext.setCheckpointDir()方法)才能启用
ReliableRDDCheckpoint。

ReliableRDDCheckpointData继承于RDDCheckpointData，RDDCheckpointData是个抽象类，其构造参数rdd表示当前的CheckpointData属于哪个RDD。cpState是对应的RDD的
Checkpoint State，由CheckpointState对象定义，是一个枚举，取值有Initialized、CheckpointingInProgress、Checkpointed三种状态，初始时是nitialized状态。cpRDD
是一个特殊的RDD，表示的是一个CheckpointRDD实例，包含了我们的Checkpointed Data。checkpoint()方法包含了保存检查点的逻辑，用于保存当前的RDD的内容，它由final关
键字修饰，说明子类不可以覆盖该方法，它的执行逻辑是：如果cpState的值是Initialized，则将其置为CheckpointingInProgress，表示正在进行checkpoint，然后调用doCheckpoint()
方法生成newRDD。在这里，doCheckpoint()是个抽象方法，ReliableRDDCheckpointData与LocalRDDCheckpointData分别都有其具体实现，在ReliableRDDCheckpointData.doCheckpoint()
方法中是通过调用ReliableCheckpointRDD.writeRDDToCheckpointDirectory()方法生成newRDD。最后将生成的newRDD赋值给cpRDD，将cpState置为Checkpointed，并调用
RDD.markCheckpointed()方法标记检查点已经保存完成。markCheckpointed()的实现比较简单，就是清除RDD原先持有的分区和依赖信息。这些东西在检查点里已经保存了一
份，不需要重复保存。

在上面的doCheckpoint()方法返回的数据类型是CheckpointRDD，它也是个抽象类，继承自RDD，并将RDD类中doCheckpoint()、checkpoint()和localCheckpoint()三个方法都使用空方法体覆盖，
因为CheckpointRDD本身是一个特殊的RDD，它并不需要再次被Checkpoint。另外，它还覆盖了getPartitions()方法和compute()方法，之所以要实现这两个方法注释中也有写，是因为如果不实现会
有bug，在这里的实现是???，它在scala.Predef中有定义：def ??? : Nothing = throw new NotImplementedError(要使用???，需要用scalastyle:off关闭静态检查)，相当于是没有实现。

普通RDD的compute()方法用于计算分区数据，在CheckpointRDD中，它的作用则是从检查点恢复数据。与RDDCheckpointData类似，CheckpointRDD也有两个子类，即ReliableCheckpointRDD
和LocalCheckpointRDD，后者实现很简单，这里主要看一下比较复杂的实现ReliableCheckpointRDD类。

ReliableCheckpointRDD中的很多方法的具体实现都是在其伴生对象中，我们先看下上面已经调用过的writeRDDToCheckpointDirectory()方法，这个方法主要用于实现检查点数据的写入。其执行
流程是：通过调用HDFS相关的Api创建检查点目录，然后在广播hadoop的配置后，调用SparkContext.runJob()方法运行一个Job，该Job执行writePartitionToCheckpointFile()方法的逻辑，将
RDD的分区数据写入检查点目录，再检查原RDD是否定义了分区器，如果定义了，就调用writePartitionerToCheckpointDir()方法将分区器的逻辑写入检查点目录。最后创建ReliableCheckpointRDD
实例，并检查它的分区数是否与原RDD的分区数相同，相同则成功返回。

在检查点数据写入到检查点目录后，如果任务出现异常需要恢复该怎么读取检查点的数据呢，这个由compute()方法来实现，在这个方法中调用了readCheckpointFile()方法，该方法使用HDFS Api打开
检查点目录下的文件(肯定的，因为是用HDFS Api写入的)，并使用SparkEnv中初始化的JavaSerializer反序列化读取出的数据，最终返回数据的迭代器，如此就恢复了现场。