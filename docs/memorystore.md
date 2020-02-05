### MemoryStore

返京的第一天，火车上都没什么人，第一次能够在春运期间的卧铺车厢里面看到6个铺位只睡了两个人，看来大家都因为疫情而不敢出门啊！虽然受到疫情影响，心情也受到
了影响，但是生活还是要继续，阅读源码这个事也要继续！

前面阅读内存相关的源码阅读了这么几天时间，看了MemoryPool及其两个子类的实现，也看了MemoryManager及其两个子类的实现，结果现在回想起来都是干了些记账的
活儿。既然它们都没真正干内存分配和释放的活儿，但是活儿总是要有人干的啊，是谁真正的干了这些呢？这时就要隆重介绍只干活不做声的内存存储类MemoryStore类啦。

当然在介绍MemoryStore类之前，先介绍一下与其同在一个文件且比较简单的MemoryEntry，这是个被sealed声明的trait，这表示的它仅能被同一个文件的其它类继承。
MemoryEntry可以理解为块在内存中的表示，其中的属性size表示块的大小，memoryMode表示的是块存储在堆内还是堆外，classTag表示该块所存储的对象类型。
点开它的实现发现它有两个子类，分别是：DeserializedMemoryEntry类和SerializedMemoryEntry类，从名字上就能看出前者是其反序列化实现，后者是其序列化实
现。在DeserializedMemoryEntry类中，memoryMode被固定为MemoryMode.ON_HEAP，也就是反序列化的块只能存储在堆内，其值是classTag类型的数组。而SerializedMemoryEntry
没有固定memoryMode，既能用于堆内也能用于堆外，其值ChunkedByteBuffer在前面分析BlockData时已经讲到过，其大小就是hunkedByteBuffer的大小。

再来看一下BlockEvictionHandler，它也是个trait，注释中讲到它从内存中淘汰掉一个块，如果可能的话会将淘汰掉的块放入到磁盘，当内存到达了其上限且需要释放
空间时被调用，且调用它时需要持有块的写锁。

在介绍完了MemoryEntry和BlockEvictionHandler后，现在可以正式开始介绍MemoryStore了。先来看它的成员属性：
  * entries：BlockId与MemoryEntry数组的映射关系，使用LinkedHashMap存储，初始容量32，负载因子0.75，按访问顺序排序;

  * onHeapUnrollMemoryMap/offHeapUnrollMemoryMap：分别存储taskAttemptId和该Task在堆内、堆外占用的UnrollMemory大小之间的映射关系;

  * unrollMemoryThreshold：在展开块之前申请的初始内存大小，由spark.storage.unrollMemoryThreshold配置项控制，默认是1MB;

此外，还有几个获取对应内存大小的方法：
  * maxMemory：可用于存储的内存总大小，单位是字节;

  * memoryUsed：包括展开内存在内的已使用的内存大小;

  * blocksMemoryUsed：除展开内存外的存储内存大小，值是memoryUsed - currentUnrollMemory;

  * currentUnrollMemory：当前展开内存使用的大小，是堆内展开内存使用大小与堆外展开内存使用大小之和;

再来看下写入数据的方法，先来看下直接写入字节的方法putBytes()：如果blockId不存在于entries，这表明该块还没有被写入，那就调用memoryManager.acquireStorageMemory()
方法申请所需的存储内存用于写入当前块，然后通过传入参数中的_bytes()函数来获取待写入的数据，获得的数据类型是ChunkedByteBuffer。根据获得的
数据创建SerializedMemoryEntry，并将其放入entries映射(其类型是LinkedHashMap，因此需要加锁保证线程安全)。写入成功返回true，写入失败返
回false。

putBytes()写入的是字节数据，而putIterator()写入的则是迭代器化的数据，也就是Iterator[T]形式的数据，有时块数据可能过大导致无法在内存中
物化和存储，为了避免OOM异常，这个方法会一边遍历迭代器一边周期性的检查内存是否有足够的空闲内存来存储。如果块被正确的物化，那么在物化过程中的
临时展开内存能够被"转换"为存储内存，所以我们可以不用申请额外的超出存储该块实际需要的内存。这个方法会被putIteratorAsValues()方法和putIteratorAsBytes()
方法调用，分别产生DeserializedMemoryEntry和SerializedMemoryEntry，这两方法最终都会调用utIterator()方法写入数据。直接来看putIterator()
方法吧：
  * 先获取一些需要用到的数据和配置。其中spark.storage.unrollMemoryCheckPeriod配置项表示的是迭代过程中检查内存是否够用的周期，默认值是
  16，也就是每展开16个元素就检查一次。spark.storage.unrollMemoryGrowthFactor配置项表示的是申请内存来展开块时的扩展倍数，默认是1.5倍。

  * 调用reserveUnrollMemoryForThisTask()方法来申请足够的内存开始展开;

  * 循环迭代块的数据，将其放入valuesHolder中，实际上是存储在了其中的SizeTrackingVector中，这个数据结构是一个只允许追加的缓存并且能够
  持续跟踪估算其内存储的数据的大小;

  * 如果到达了迭代过程中检查内存是否够用的周期，并且vector的大小超过了当前的展开内存的阈值，就需要再次调用reserveUnrollMemoryForThisTask()
  方法申请新的展开内存，申请大小是当前vector的大小乘以扩展倍数再减去当前的展开内存的阈值，如果申请成功则更新阈值;

  * 在所有的数据都被展开后，keepUnrolling被赋值为true，表示展开成功。此时可能块的实际内存使用大小比展开内存会要稍大一些，所以如果需要的话，
  会尝试再次调用reserveUnrollMemoryForThisTask()来申请额外的内存以确保有足够的内存存储该块。

  * 在所有的数据都被展开后，会调用releaseUnrollMemoryForThisTask()方法来释放掉多余的展开内存，并将其还给存储内存(也就是将展开内存转换
  为存储内存);

  * 如果一切顺利，会将块ID与MemoryEntry的对应关系放入到entries中，并返回Right，否则返回Left;

再来看下evictBlocksToFreeSpace()方法吧，这是一个淘汰块来释放一些空间以存储特定新块的方法：其执行逻辑如下：
  * 遍历entries映射中的块，如果块能够被淘汰(块能够被淘汰是指memoryMode相同，且blockId对应的块不是RDD的块);

  * 为块加写锁，以确保当前正在读取的块不会被淘汰掉，记录下将要被淘汰的块的blockId;

  * 如果将要被释放的空间大小已经达到了希望获得的空间大小，则调用dropBlock()方法真正的移除这些块，dropBlock()方法中最终还是调用了blockEvictionHandler.dropFromMemory()
  方法，该方法可能将块转存到磁盘，也可能被彻底的删除。如果是前者就释放其上的锁，如果是后者则表示块已经不存在，调用blockInfoManager.removeBlock()
  方法删除块的元数据信息，最后在finally中将未处理的块解锁;

  * 如果将要被释放的空间大小没有达到希望获得的空间大小，新的块不会被写入，也不会执行任何淘汰操作;

至此，总算把内存存储相关的源码阅读完毕啦！！！