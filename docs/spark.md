### Spark

RDD：弹性分布式数据集，是Spark中最重要的概念之一，它是一个可以并行的、可容错的数据集合。

Application：简单来说，用户每次提交的所有代码就是一个Application。

Job：一个Application可以分为多个Job，每次触发一个final RDD的计算就是一个Job。

Stage：一个Job可以分为多个Stage，它根据Job中RDD的依赖关系来分，每遇到一个宽依赖就会划分成一个新Stage。

Task：是最小最基本的计算单位，一般是数据的一个分块是一个Task，大小为128M。

