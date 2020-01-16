### RDD操作

Spark 中所有的 transformations 都是 lazy（懒加载的），因此它不会立刻计算出结果。相反，他们只记得应用于一些基本数据集的转换（例如：文件）。
只有当需要返回结果给驱动程序时，transformations 才开始计算。这种设计使 Spark 的运行更高效。例如，我们可以了解到，map 所创建的数据集将被用
在 reduce 中，并且只有 reduce 的计算结果返回给驱动程序，而不是映射一个更大的数据集。

默认情况下，每次你在 RDD 运行一个 action 时，每个 transformed RDD 都会被重新计算。但是，您也可用 persist（或 cache）方法将 RDD
persist（持久化）到内存中；在这种情况下，Spark 为了下次查询时可以更快地访问，会把数据保存在集群上。此外，还支持持续持久化 RDDs 到磁盘，
或复制到多个结点。cache和persist都是控制算子，它们的区别是cache是persist的特例，cache是只能将RDD持久化到内存，而persist则可以传递StorageLevel，
选择将RDD持久化的介质，此外还有另外一种控制算子叫做checkpoint，它也能将RDD持久化，它与前两者的区别是不仅能将RDD持久化，还能切断RDD之间的依赖关系。

在计算期间，一个任务在一个分区上执行，为了所有数据都在单个 reduceByKey 的 reduce 任务上运行，我们需要执行一个 all-to-all 操作。它必须
从所有分区读取所有的 key 和 key对应的所有的值，并且跨分区聚集去计算每个 key 的结果 - 这个过程就叫做 shuffle。

RDD的依赖关系可以分为两种：
(1) 窄依赖：父RDD的每个分区最多被其子RDD的一个分区所依赖，也就是说子RDD的每个分区依赖于常数个父分区，子RDD每个分区的生成与父RDD的数据规模
无关.

(2) 宽依赖：父RDD的每个分区被其子RDD的多个分区所依赖，子RDD的每个分区的生产与父RDD的数据规模相关.

区分宽依赖与窄依赖的原因是：窄依赖关系的RDD在集群的节点的内存中可以以流水线(pipeline)的方式高效运行.

创建RDD:
(1) 对驱动程序中的集合进行并行化的处理，包括：makeRDD、parallelize，区别是makeRDD可以指定每一个分区preferredLocations参数
(2) 读取外部存储系统(HDFS、Hbase、Hive等)，包括：文本文件、SequenceFile、Avro、Parquet等，其中textFile支持.gz格式的压缩文件读取,需要注意的是，
读取文件时，传递的分区参数为最小分区数，但是实际上不一定是这个分区数，实际的数字取决于hadoop读取文件时的分片规则。
(3) 基于已有RDD的转换

RDD的执行操作：first、count、collect、take

