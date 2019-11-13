### Task调度

在[Spark源码阅读5：Stage的提交](./stagesubmit.md)已经说到，最终stage被封装成TaskSet，并使用taskScheduler.submitTasks进行提交。