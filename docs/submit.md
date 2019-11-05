### Spark任务提交

在将Spark应用程序提交到集群运行时，会使用spark-submit脚本，该脚本比较简单，分析该脚本知道其实最终调用的是spark-class脚本，传入的参数是SparkSubmit
及其他用户传入的参数。在spark-class中，首先会使用load-spark-env.sh加载spark的环境变量信息、定位spark jars文件等，然后调用org.apache.spark.launcher.Main
正式启动org.apache.spark.deploy.SparkSubmit的执行。