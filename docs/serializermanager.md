### SerializerManager

序列化和反序列化渗透到了大数据处理的各个阶段和每个角落，在Spark Core中同样也不例外。此外，SerializerManager除了负责序列化操作外，还涉及到一部分
压缩和加解密的工作。它的主构造方法接收三个参数：
  * defaultSerializer：默认的序列化器，它已经在SparkEnv初始化时创建好了，类型是JavaSerializer。

  * conf：即Spark的配置项SparkConf。

  * encryptionKey：加密时使用的密钥，可选，并且只有当存在加密密钥时才会启用加密。

SerializerManager中的成员属性有以下这些：
  * kryoSerializer：使用google kryo序列化库的序列化器，效率比JavaSerializer高，但是原生支持的类型较少，如果需要使用自定义类型，需要提前进行注册。

  * stringClassTag：String类的类型标记，由于Java和Scala中的泛型会在编译器进行类型擦除，所以类型标记在Java和Scala中用来在运行期指定无法识别的泛型类型(这
  感觉有点类似于Flink中的TypeInformation)。

  * primitiveAndPrimitiveArrayClassTags：Scala基本类型及其对应数组的所有类型标记，基本类型共有八种，分别是Boolean、Byte、Char、Double、Float、Int、
  Long、Null、Short八种。

  * compressBroadcast：是否对广播变量进行压缩，对应的配置项是spark.broadcast.compress，默认为true，表示需要进行压缩。

  * compressShuffle：是否对Shuffle过程中的输出数据进行压缩，对应的配置项是spark.shuffle.compress，默认值true，表示需要进行压缩。

  * compressRdds：是否对序列化的RDD分区数据进行压缩，对应的配置项是spark.rdd.compress，默认为false，表示不压缩。

  * compressShuffleSpill：是否对Shuffle过程中向磁盘溢写的数据进行压缩，对应的配置项是spark.shuffle.spill.compress，默认为true，表示需要进行压缩。

  * compressionCodec：使用的压缩编解码器，它是CompressionCodec特征的实现类，并且它会延迟初始化。

SerializerManager获取序列化器时，会先调用canUseKryo()方法来判断需要序列化的对象是否是Scala中的八种基本类型或String类型，如果是则获取KryoSerializer，
否则使用默认的JavaSerializer。获取序列化器的getSerializer()的方法有两种重载，其中带有keyClassTag和valueClassTag两个入参的重载方法专门用于确定Pair RDD
在Shuffle过程中的序列化器。

SerializerManager提供了多种方法来对输入流和输出流进行包装，将它们转化为压缩或加密的流。在转化为加密的流时，如果encryptionKey存在的话，通过调用wrapForEncryption()
方法可以将流转化为加密的流。如果存储块ID对应的数据类型支持压缩，可以调用wrapForCompression()方法可以将流采用指定的编解码器进行压缩。

SerializerManager对序列化器Serializer的serializeStream()及deserializeStream()方法进行了一定的封装，其序列化方法既可以直接序列化为流，或根据值的ClassTag序列化
为ChunkedByteBuffer，即分块的字节缓存。而其反序列化方法则直接返回值类型的迭代器，并且当存储块ID的类型是StreamBlockId(Spark Streaming中用到的块ID)时，将不会自动
判断应该使用哪种序列化器，而是完全使用用户指定的类型。

现阶段，SerializerManager压缩的实现主要是依靠CompressionCodec，而这个特征只定义了两个方法(也就是compressedOutputStream和compressedInputStream)，所有的逻辑
都实现在它的伴生对象中。从伴生对象中，我们可以看到，目前Spark支持四种压缩方式，分别是：lz4、lzf、snappy、zstd，可以通过配置项spark.io.compression.codec指定，其中
lz4是默认的压缩方式，由常量DEFAULT_COMPRESSION_CODEC指定，而createCodec()方法会获得Codec短名称对应的具体类名，然后通过反射的方式创建对应的实例。