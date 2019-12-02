### Spark运行架构

![Spark运行架构图](../image/spark-runtime.png "Spark运行架构图")

其中，Driver负责作业逻辑的调度和任务的监控，资源管理器负责资源的分配和监控，根据部署模式的不同，启动和运行的物理位置也有所不同。在Client模式下，Driver模块
运行在Spark-Submit进程中。Cluster模式下，Driver的启动过程与Executor类似，运行在资源调度器分配的资源容器内。