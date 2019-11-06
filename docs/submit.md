### Spark任务提交

在将Spark应用程序提交到集群运行时，会使用spark-submit脚本，该脚本比较简单，分析该脚本知道其实最终调用的是spark-class脚本，传入的参数是
SparkSubmit及其他用户传入的参数。在spark-class中，首先会使用load-spark-env.sh加载spark的环境变量信息、定位spark jars文件等，然后调用
org.apache.spark.launcher.Main正式启动org.apache.spark.deploy.SparkSubmit的执行。

在submit方法中，包含两个步骤：1.基于集群管理器和部署模式为运行的主子类设置适当的类路径、系统属性和应用参数以准备运行环境。2.使用运行环境去
调用主子类的主方法。