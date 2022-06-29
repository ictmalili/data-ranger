# 阅读材料
https://flink.apache.org/
https://nightlies.apache.org/flink/flink-docs-release-1.15/zh/docs/learn-flink/overview/ 
https://www.youtube.com/watch?v=ZU1r7uEAO7o&list=PLaDktj9CFcS9YAaJ4bKWMWpjptudLr782

# Flink Basic
Flink是由公司Ververica创建的。流处理框架
低延迟、高并发、可容错，最关键的是stateful。
![Flink Definition](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20Flink%20Definition.png)

# 批处理 vs 流处理
## 批处理的缺陷：
1. 数据处理延迟。因为先到达数据必须要等到batch处理点才进行处理
2. 人为引入spike workload。因为攒了一部分数据集中处理
3. 可能带来错误。如果批处理是按确定时间分批的话，有可能一个连续的操作（类似事务）被打散在两个不同的批里，进而影响正确性。解决方法是跨批处理，但何时跨、如何跨好似一个问题. 


## 流处理
-> 可以处理源源不断的数据流，流处理代码持续运行而且有自己的状态（state），有数据过来时直接运行，每次处理一条记录，然后生成一系列流。
![Stream Processing](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20stream%20processing.png)

## 分布式流处理
-> 可以跨多个实例/节点部署，每个实例处理其中一部分数据。输入方可以调用key方法，决定这条记录放到哪个实例去运行。
![Distributed Stream Processing](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20distributed%20stream%20processing.png)

## 流处理的应用场景
1. Streaming ETL
2. Analytics （ad-hoc，SQL）
3. Event-Driven Application
流处理因为有stateful，可以在某些场景下替换应用必须把数据写入数据库，进行一些操作的调用，然后读取出来的操作。
![Event Driven Application Use Case](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20event%20driven%20application%20use%20case.png)

# Flink的两个重要概念： Dataflow 和 Snapshot
## Dataflow
1. DataStream API里可以定义接口，比如Source、Transformation、Transformation、Sink
2. Flink会把用DataStream API定义的程序转换成Streaming workflow，比如Source->flatMap()/KeyBy()->process->sink
3. Flink会把Streaming workflow转换成Execution Graph，比如跨多个节点分布，每个节点用自己的state。
![Data Stream API & Streaming workflow](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20data%20stream%20API%20%26%20Streaming%20workflow.png)
![Execution Graph](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20execution%20graph.png)

Flink也可以支持对SQL/Table 的API （是不是类似于Kinesis Data Analytics？）
![SQL/Data API](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20SQL:Table%20API.png)

## Snapshot
打快照
Flink打快照时是打的Consistent Distributed Snapshots。通过在Event流加入一个特殊的marker（checkpoint barriers），当checkpoint barriers流到有State的operator时，Operator会把自己的状态放入到fault-tolerant storage，比如HDFS、S3。当所有有状态的opeartor都把state持久化以后，这个快照就完成了。

故障恢复/rollback
Checkpoint barrier有自己在datastream中的offset，所以按照offset进行恢复即可。
![Checkpoint rollback](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-checkpoint%20rollback.png)

# Use Case
做event-driven application
1. 分布式，fault-tolerant
2. 自带状态，无需写DB （这里的state是值每个key都有自己的state）
3. 水平扩展
4. 可以并行执行，但没有concurrency issue。因为数据都按照ID打到相应的实例上了。（问题：这里打的时候时对hash值按照range划分的）
![Use Case](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20use%20case.png)

# Flink架构
## Cluster模块
有一个Job Manager(coordinator)，多个Task manager(worker)。Job Manager负责将job deploy到task manager上，task manager会把自己的状态以及heartbeat同步到Job Manager。（类似于Hadoop）
![Flink Cluster components](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20Flink%20cluster%20components.png)

打快照时，Job manager会trigger task manager进行快照，task manager会把信息存在持久化snapshot store里。（HDFS/S3）
![Flink Snapshot Processing](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20snapshot%20processing.png)

资源管理有两种方式
1. 静态资源管理。 每个task manager向job manager发送注册信息。job manager收到client请求后，会根据task manager的注册信息生成execution graph（主要是task manager数目，决定将数据拆成几份）
![Static Resource Provisioning](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20static%20resource%20provisioning.png)
![Static Resource Provisioning - 2](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20static%20resource%20provisioning%20-%20generating%20execution%20graph.png)
2. 动态资源管理。client向job manager提交任务，job manager里的resource manager向YARN/K8S请求资源，拿到资源后会去部署task manager。
![Dynamic Resource Provisioning](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20dynamic%20resource%20provisioning%20-%20yarn:k8s.png)
