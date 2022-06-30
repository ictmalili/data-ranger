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

## 资源管理
资源管理有两种方式
1. 静态资源管理。 每个task manager向job manager发送注册信息。job manager收到client请求后，会根据task manager的注册信息生成execution graph（主要是task manager数目，决定将数据拆成几份）
![Static Resource Provisioning](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20static%20resource%20provisioning.png)
![Static Resource Provisioning - 2](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20static%20resource%20provisioning%20-%20generating%20execution%20graph.png)
2. 动态资源管理。client向job manager提交任务，job manager里的resource manager向YARN/K8S请求资源，拿到资源后会去部署task manager。
![Dynamic Resource Provisioning](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20dynamic%20resource%20provisioning%20-%20yarn:k8s.png)

## 部署架构
![Flink Deployment Model](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20Flink%20Deployment%20Model.png)

## Job Manager的HA
Job Manager是单点，HA是通过将Job metadata放在Zookeeper里完成。如果Job Manager出现故障，新启动的可以从Zookeeper里拿信息
![Job Manager HA](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20Flink%20Job%20Manager%20High%20Availability.png)

# Flink对时间的处理
主要是Event Time和Process Time。Event Time是指Event在设备上生成的时间，处理起来是有可以保证正确结果的。processing time每次处理时间都不一样。
![Notion of Time](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20notion%20of%20time.png)
基于Event Time处理带来的一个问题是Event到达Flink Opeartor处理的时间可能是乱序的。就需要考虑如何保证一定时间内的event能够会攒在一起处理
Flink有time window的概念，先根据time分成几部分，比如10s为一个time window，来把event打散。

但是怎么保证一个timewindow里的event完全到齐了？引入了watermark的概念。watermark到来，在它以前的event可以认为都处理完了。
watermark可以人为加入，也可以通过event id来设置，比如设定当前eventid-6就是watermark。
![WaterMark](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20watermark.png)

watermark引入了，但也不能保证在它之前的event都处理完了。但如果后续的late event怎样去处理？几种不同
方案：可以丢弃，也可以选允许最大延迟的多大
![WaterMark and Late Events](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20late%20events.png)

![Time Summary](https://github.com/ictmalili/data-ranger/blob/master/Flink/graph%20-%20time%20impact.png)

# Exactly Once的处理
Flink Exactly-once实现原理解析 https://zhuanlan.zhihu.com/p/266620519
https://blog.51cto.com/u_14222592/2892647

## 分布式快照/Checkpoint （Barrier+异步+增量）

* checkpoint vs savepoint： checkpoint是系统自动trigger，每隔一段时间打一次，比如10s。savepoint是用户/应用程序trigger的

* checkpoint的实现通过在Event flow里引入Barrier（特殊的event marker），当barrier流到不同的operator时各自进行状态的持久化，大家都完成以后认为全局提交。

* 分布式快照是为了在多个不同有状态的operator之间（可能是一组节点对不同数据子集执行同一个operator，或者是多个operator）达到全局一致，才引入的。

* 如果一个flink对应多个流，会快流等慢流，等大家barrier都到了再把状态写出去。

## Transaction Sink 两阶段提交
实现思想：构建的事务对应着 checkpoint，等到 checkpoint 真正完成的时候，才把所有对应的结果写入 sink 系统中

实现方式

1）预写日志（WAL）：把结果数据先当成状态保存，然后在收到 checkpoint 完成的通知时，一次性写入 sink 系统。
缺点：做不到真正意义上的Exactly-once，写到一半时挂掉可能重复写入。

2）两阶段提交（2PC）：
* 对于每个 checkpoint，sink 任务会启动一个事务，并将接下来所有接收的数据添加到事务里
* 然后将这些数据写入外部 sink 系统，但不提交它们，这时只是“预提交”
* 当它收到 checkpoint 完成的通知时，它才正式提交事务，实现结果的真正写入
* 这种方式真正实现了 exactly-once，它需要一个提供事务支持的外部 sink 系统。


Flink 中两阶段提交的实现方法被封装到了 TwoPhaseCommitSinkFunction 这个抽象类中，我们只需要实现其中的beginTransaction、preCommit、commit、abort 四个方法就可以实现“精确一次”的处理语义。

1. beginTransaction，在开启事务之前，会在目标文件系统的临时目录中创建一个临时文件，在处理数据时将数据写入这个文件里面。
2. preCommit，在预提交阶段，将内存中缓存的数据刷写（flush）到文件，然后关闭文件。还将为属于下一个检查点的任何后续写入启动新事物
3. commit，在提交阶段，将预提交写入的临时文件移动到真正的目标目录中，这代表着最终的数据会有一些延迟；
4. abort，在中止阶段，我们删除临时文件。


## Exact Once
是通过分布式快照+两阶段提交实现的
（question：transaction ID怎么生成的？）

两阶段提交的实现：
1. 一旦 Flink 开始做 checkpoint 操作，那么就会进入 pre-commit 阶段，同时 Flink JobManager 的Coordinator会将检查点 Barrier 注入数据流中 ；
2. 当所有的 barrier 在算子中成功进行一遍传递，并完成快照后，则 pre-commit 阶段完成；
3. 等所有的算子完成“预提交”，就会发起一个commit“提交”动作，但是任何一个“预提交”失败都会导致 Flink 回滚到最近的 checkpoint；
4. pre-commit 完成，必须要确保 commit 也要成功，上图中的 Sink Operators 和 Kafka Sink 会共同来保证。

Exact Once对外部系统有要求（source和target），不一定所有都能实现exact once。他要求source能够重新传数据（数据有一定存放能力，可以重放），target能够实现幂等（相同数据多次执行结果一样）。

Flink目前支持的精确一次source列表如下表所示，你可以使用对应的connector来实现对应的语义要求：

数据源	语义保证	备注

Apache kafka	exactly once	需要对应的Kafka版本

AWS Kinesis Streams	exactly once	 

RabbitMQ	at most once（v0.1）/ exactly once（v1.0）	 

Twitter Streaming API	at most once	 

Collections	exactly once	 

Files	exactly once	 

Sockets	at most once	 

如果需要实现真正的“端到端精确一次语义”，则需要sink的配合，目前Flink支持的列表如下所示：

写入目标	语义保证	备注

HDFS rolling sink	exactly once	依赖 Hadoop 版本

Elasticsearch	at least once	 

Kafka producer	at least once / exactly once	需要 Kafka 0.11 及以上

Cassandra sink	at least once / exactly once	幂等更新

AWS Kinesis Streams	at least once	 

File sinks	at least once	 

Socket sinks	at least once	 

Standard output	at least once	 

Redis sink	at least once

# Off-Cycle 讨论
## 两阶段提交
两阶段提交 （need confirm）
1. 参与者完成本地操作，进入prepare模式，发送给协调者
2. 协调者收到所有参与者的消息，进入commit模式，会把自己本身的transaction状态置为commit，然后给各个参与者发指令。
3. 参与者收到协调者发送指令，执行本地commit模式。
4. 如果参与者在第2步以后失败，故障重启后应该会向协调者要状态，再继续进行本地commit。（协调者为中心状态存储者）

Greenplum在故障恢复时，没有重新向协调者要状态的过程，协调者会重建Gang，给各个参与者发送commit信息。 （@chunling，please confirm）

## PostgreSQL的WAL日志写入问题
PostgreSQL中有WAL，防止数据页由操作系统写出时不完全（数据页大小大于操作系统向磁盘写数据单元大小），在每次checkpoint以后的新WAL里会加入更改数据页的全部信息。如果频繁更改，频繁刷checkpoint，会带来写放大问题（GaussDB就是这么做的）

另外一个问题是：WAL写数据页就不担心写坏吗？比如事务T1和事务T2都是写数据页P1，事务T1在前，在flush以后数据页P1写入成功，事务T2在后，写数据页P1如果发生失败，会不会带来数据页1的后续读取出问题？

Answer：应该不会，如果页面在前面没有整个页的checksum的话。每个事务可以有自己的元信息，事务T1的元信息成功写入了。WAL是append-only的，所以事务T2的信息是在后面的，出错也是T2出错。

再一个问题：如果一个调用fsync中部分成功，部分失败，后续故障恢复以后会不会有问题？因为fsync并不是原子性操作，而磁盘原子单位为512字节，fsync会把要写的内容乱序分发，并行写出。这样有可能会带来page的前面一个512字节失败，后面的几KB是成功的。如果fsync对应的是事务的组提交，会怎样？如果后续数据库需要故障恢复，再写同一个page会怎么办？

Answer：fsync失败，所有事务都失败，都回滚，问题不大。如果后续数据库故障恢复，遇到page前面的失败以后，应当把WAL log文件对应的后续的所有页面都置0，这样可以避免这个情况出现：恢复后新的事务正好塞满512字节对应的空洞，之前的成功的几KB能够读，但他们对应的事务已经被回滚。带来的问题是，如果WAL log（redo log）较长，置空操作耗时。 这也没办法。


# 下次讨论主题
继续探讨流失数据处理、流批一体等架构 （2022年7月13日）
1. Lambda & Kappa架构
2. 流批一体、流处理特点
3. yMatrix/Greenplum在流处理中的位置

