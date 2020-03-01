### SparkConf

SparkConf负责管理Spark的配置相关项，是初始化SparkContext的前提，通过它可以灵活的配置任务运行的各种参数，使程序更快更好的运行。

首先，看下SparkConf类的构造方法。首先它通过import语句从SparkConf的伴生对象中导入配置项，主要用于管理过期的、与旧版本兼容的配置项和日志输出。
需要注意的是，Scala中没有Java中的static的概念，类的伴生对象中维护的成员和方法就可以看作是类的静态成员和静态方法。

SparkConf类的主构造函数参数loadDefaults(在Scala中主构造函数和类的定义融合在一起，它的参数列表放到了类名的后面，它的方法体就是除去字段和方法声明语句后的整个类体)，
它标识是否要从Java系统属性也就是System.getProperties()取得属性并加载与Spark相关的配置(所谓与Spark相关，就是属性key以spark.开头的属性配置)。

SparkConf内部采用settings存储所有配置，它是ConcurrentHashMap类的实例，之所以用ConcurrentHashMap肯定是考虑到了并发环境下的线程安全问题，其键值类型
都是String，也就是表示所有的配置项都是以字符串形式存储。

可以通过三种方式设置Spark的配置项：
  * 直接调用set方法设置，这是最常见的设置方法，SparkConf提供了多种重载的set方法，但是最终都调用到了有三个参数的set(key: String, value: String, silent: Boolean)
  方法，在这个方法中我们可以看到配置项的key和value都不能为null，否则会抛出NPE错误，silent表示的是如果给定的配置项已过时是否要打印警告，并且所有set方法及其重载方法都返回
  this，因此能够通过链式调用简化代码。
  * 通过系统属性加载，如果设置SparkConf类的主构造函数参数loadDefaults为true，那么SparkConf会从Java系统属性中加载配置项，如果调用无参的辅助构造方法new SparkConf()，
  也会将loadDefaults设置为true，java系统属性可以通过System.setProperties()方法在程序中动态设置。在具体实现上，它是通过Utils通用工具类取得系统属性，过滤出以"spark."
  开头的属性，调用set方法设置配置。由于这个设置是一次性初始化的，所以可以用set方法来覆盖它们。
  * 克隆SparkConf，SparkConf类继承了Cloneable(这是个trait修饰的特征，有点类似于java中的接口，但功能更多)，并覆盖了clone()方法，它是可以深度克隆的。虽然已经使用
  ConcurrentHashMap结构来保证并发时的线程安全，但高并发场景下的锁机制还是会带来性能问题，我们可以通过克隆SparkConf的方式让多个组件获得同样的配置。

SparkConf也提供了一些方法来快速设置常用的配置项，例如在大数据的HelloWorld--WordCount程序中，通过setMaster()和setAppName()来进行master和appName的
设置，它们最终也会调用set()方法。

可以通过get方法获取配置，get方法同样有很多重载的实现，在实现上它会同时检查过期的配置(getDeprecatedConfig()方法是在SparkConf的伴生对象中定义的)，并使用Scala的Option对
返回结果进行包装，可以实现对为空和不为空的情况进行处理。

SparkConf中还有一个validateSettings()方法，它用来对不合法的配置或过期配置进行校验，并对不合法的配置抛出异常，这个方法不是幂等的，它可能会对配置对象进行转换以将过期的配置
转换为支持的配置。
