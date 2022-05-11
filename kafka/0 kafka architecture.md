# Articles
1. [Official doucument for kafka](https://kafka.apache.org/documentation/#introduction)
2. [中文版文档](https://kafka.apachecn.org/documentation.html)

# Youtube Videos
1. [Kafka 入门视频](https://www.youtube.com/watch?v=XFqm_ILuhs0&list=PLt1SIbA8guusxiHz9bveV-UHs_biWFegU)
2. [Kafka Streaming](https://www.youtube.com/watch?v=y9a3fldlvnI)
3. [Plusar Overview](https://www.youtube.com/watch?v=7h7hA7APa5Y)
4. [Pulsar](https://www.youtube.com/watch?v=vlU9UegYab8)
5. [Pulsar Basic Concepts](https://www.youtube.com/watch?v=ed5zxfvlT-M)
6. [Kafka vs Amazon SNS/SQS](https://www.youtube.com/watch?v=ZI5CDsob6i0)


# 讨论的Summary 
## Kafka Scenario
![企业应用场景](https://github.com/ictmalili/data-ranger/blob/master/kafka/4%20Kafka%20in%20the%20Enterprise%20Architecture.png)

* Decouple/缓冲/动态处理
* Sharding OLTP -> Kafka -> OLAP
* IOT -> MatrixGate -> OLAP IOT数据拿过来，批量存放，在放到AP系统中
* Kafka的consumer是pull的模式，不是push

## Kafka Fundamental
Concepts: 
* Pub/Sub 
* Topics -> Partitions
* Producers -> Consumers
* Brokers

![基本组成](https://github.com/ictmalili/data-ranger/blob/master/kafka/1%20Kafka%20Ecosystem%20-%20Kafka%20Core.png)
![Brokers & Topics](https://github.com/ictmalili/data-ranger/blob/master/kafka/6%20Brokers%20%26%20Topics.png)
![Producers](https://github.com/ictmalili/data-ranger/blob/master/kafka/9%20Producers.png)
![Consumers](https://github.com/ictmalili/data-ranger/blob/master/kafka/11%20Consumer%20Groups.png)

## Kafka vs 时序数据库
时序数据库的查询处理要更加复杂，Kafka上也在向这个方向走，比如做KSQL，但可能不容易做这么成熟

## Kafka vs 消息队列 RabbitMQ/AWS SQS/AWS SNS
1. RabbitMQ是基于队列的，比如有严格的队列处理，对于未处理成功的消息，可以有dead-letter queue来存放。如果消息取出成功，就自动从队列里删除了。Kafka是基于Log based的。每个partition的数据是排序的，是appendonly增长的。数据被consumer消费以后，还是可以继续存在Kafka里的。

2. SQS可以多个consumer对应同一个队列。 比如SQS+ EC2 in Auto Scaling group。多个consumer从一个队列中取数据，取完一个去处理。Kafka可以有consumer group，consumer group里有多个consumer，但是consumer和partition是1对N的关系。即每个consumer去1个或多个partitions中取数据。当然可以设置不同的consumer group，读相同数据。

3. Kafka/Kinesis的处理时延更低，可以做近实时的处理，但Queue不一样，有可能堆积。所以后端消费者要及时scale

4. Kafka可以在partition内保证数据的有序性，SQS一般是无序的，可以通过设置FIFO来保证有序。

## Kafka vs Pulsar
Pulsar包括三部分：zookeeper（元信息）、Pulsar Server（计算层），apache bookkeeper（存储层）。类似于Greenplum和Snowflake的区别。

好处：
1. 可以存储更久的数据
2. 可以进行独立的扩展
3. 可以实现多租户。 
4. Pulsar的book keeper可以上面套单独的SQL封装，被查询。

![Pulsar](https://github.com/ictmalili/data-ranger/blob/master/kafka/26%20Pulsar%20Architecture.png)
## 有用的截图看本文件夹下其他截图
