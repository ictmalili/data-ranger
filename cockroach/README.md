# Book
1. https://www.cockroachlabs.com/guides/oreilly-cockroachdb-the-definitive-guide/

# Articles
1. [Official architecture overview for cockroach](https://www.cockroachlabs.com/docs/stable/architecture/overview.html)


# Papers
1. [CockroachDB: The Resilient Geo-Distributed SQL Database](https://dl.acm.org/doi/pdf/10.1145/3318464.3386134)

# 1月26日讨论总结


## Left Questions on 1月12日 
### 1. Aurora的读节点如何防止脏读，如何保证在buffer替换出去还能读到数据？
Aurora写节点会把VDL和正在运行事务列表发给读节点，读节点在apply redo log时只会apply小于等于VDL的redo log record，所以不会有脏读问题。buffer替换出去，只要读节点的read view还在，存储层的undo不会删除，所以读节点仍然可以从存储层拿到信息

### 2. Snapshot隔离会出现write skew的问题
比如值班室比如保证一个人在周末要值班，但现在有2个人在周末available。同时两个事务出现，拿到snapshot，发现有2个人空闲，都指派一个人去做了额外的事情，因为两个事务拿到的snapshot信息一样，所以可以执行成功。但都执行完以后，会发现没有人值班，出现错误。 -》 这就是write skew问题
解决方案：PostgreSQL有SSI（Serializable Snapshot Isolation），将snapshot串行化，进行冲突管理，会在上面情况让其中一个事务abort。

## CockRoachDB 讨论
### 1. LeaseHolder与Raft Leader的异同
LeaseHolder是在distribution layer，Raft leader在replication layer，LeaseHolder类似于Executor的角色，负责将Range的读写操作发送给Raft Leader
LeaseHolder与Raft Leader都是RAFT协议，两者通常在同一节点，偶尔情况可能不在同一节点，推测可能和两个不同RAFT协议的时间间隔有关


### 2. CockRoachDB的优化的两阶段提交
CockRoach还是两阶段提交，但是对于client端来说，当底层存储写入(writeIntent)返回成功以后即可COMMIT，所以是一阶段提交，但后续将写入变成永久（resolveIntent）还是要做的，只不过是异步处理

### 3. 分布式系统如何判断alive？如果有1000个executor，每个都要判断各自是否能够成功连接，要建立1000*999/2个1-1连接来进行heartbeat吗？
可以考虑使用单独的存储点，比如zookeeper来存放1000个节点的健康信息，有分布式的存放，而不是单一master节点成为瓶颈
节点和节点之间的网络联通性如果执行过程中失败，只能让query fail或者重试

### 4. CockRoachDB的meta range
meta range有不同层级，meta0存放在所有节点，指向meta1，meta1是大范围划分，可以切到meta2，meta2再导到实际的key/value pair
meta1和meta2应该也是RAFT协议存放的，可以看做是特殊的range
meta信息有多大？ range大小最大是512MB，所以meta的大小由range的数目决定。如果一个表只有主键，range数目可以由表的大小/512MB决定，但如果表里还有其他索引，索引可以当作特殊表，也需要有额外的range存放。


## 待讨论问题列表： 
1. writeIntent和resolveIntent底层的实现，具体操作是什么？tuple级别加tag？还是其他方式？
2. TiDB&CockroachDB底层是key/value存储，在云厂商上部署不友好？怎么实现的？ 
3. TiDB和CockroachDB为什么底层是range存放，不是hash存储？
4. 为什么串行化隔离级别需要考虑读写的并发冲突
5. HLC
6. Raft 底层是传的rocksdb（LSM）的WAL？
7. rocksdb难道真的每个KV写入都fsync？
8. indx是怎么实现的？因为index的key很可能不在range里面。支持btree index only？
9. 表的元数据也是在rocksdb？怎么防止元数据修改的时候冲突？和表数据冲突解决一样（线性化级别然后client retry来解决冲突，i.e.无锁）？

## Others
(optional) PosgreSQL 的SSI （Serializable Snapshot Isolation）的paper可以作为以后研究的candidate


# 2月23日讨论总结

## Left Questions on 1月26日 
### 1. writeIntent和resolveIntent底层的实现，具体操作是什么？tuple级别加tag？还是其他方式？
writeIntent的底层实现是会产生两个key/value pair，其中一个是真正的key value，另外一个是描述writeIntent。第一个key/value pair的key带时间戳，第二个不带，value里会描述是writeIntent，并指向第一个（真正的）key/value pair。在这个Intent被resolve之前，真正的key/value pair对其他请求是不可见的。在resolve intent的时候，第二个key/value会被删除，所以不会影响真正key/value pair 的使用

### 2. TiDB&CockroachDB底层是key/value存储，在云厂商上部署不友好？怎么实现的？
可能是基于K8S使用类似于EBS. 比S3存储贵，但是更便捷和性能好。

### 3. TiDB和CockroachDB为什么底层是range存放，不是hash存储？
 推测原因一：hash partition更偏好OLAP(e.g. distribution key的hash join）；会有rebalance overhead (e.g. when expand).
   range适合简单range条件的OLAP，单机就搞定，性能好。 
 推测原因二：在进行segment split/merge 处理时，CockroachDB只需要对一个或者两segment进行操作。不会影响其他segment。而hash partition增加一个segment需要更改hash算法，影响较大，而且较难做到一变二。
 
### 4.  为什么串行化隔离级别需要考虑读写的并发冲突
 放到下一轮单独讨论。
 https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/
 PostgreSQL的串行化隔离级别有相关论文要看一下。
 
### 5. HLC
 单独一个session讨论
 
### 6. Raft 底层是传的rocksdb（LSM）的WAL？
No. Raft底层传输的是单独的raft log，但是现在似乎raft log会写rocksdb，会有写放大。不知道Pebble解决这个问题否。 WAL是RocksDB自己收到put请求以后做的两阶段的处理，先写WAL，再写key/value。是为了保证单个节点可以实现故障恢复

### 7. rocksdb难道真的每个KV写入都fsync？
是以事务为单位的。每个rocksdb事务都fsync

### 8. index是怎么实现的？因为index的key很可能不在range里面。支持btree index only？
index是单独的表，表里可以包含index的column，主键，还可以指定额外的列（很多数据库都是这么做的）。是全局排序的。LSM排序，index实现会很简单。

### 9.表的元数据也是在rocksdb？怎么防止元数据修改的时候冲突？和表数据冲突解决一样（线性化级别然后client retry来解决冲突，i.e.无锁）？
表的元数据也在rocksdb. 元数据修改冲突单独一个session讨论，可以看MySQL 的online DDL等。另外，Google F1对DDL进行了分阶段操作，在阶段和阶段之间可以并行运行DML语句。


## Followup

三个session：
1. 串行化隔离级别的实现 https://www.drkp.net/papers/ssi-vldb12.pdf   <下一次讨论topic>
2. HLC （hybrid-logical clocks）  https://cse.buffalo.edu/tech-reports/2014-04.pdf
3. DDL解决办法。https://research.google.com/pubs/archive/41344.pdf
