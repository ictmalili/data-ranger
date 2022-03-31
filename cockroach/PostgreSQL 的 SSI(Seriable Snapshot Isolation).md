# Paper 
Seriable Snapshot Isolation in PostgreSQL https://www.drkp.net/papers/ssi-vldb12.pdf 

# 2022年3月9日讨论总结
## 为什么需要SSI？
### 1. SI 不能解决实现完全串行化，RW-antidependant的问题不能解决。
* RW-antidependant： T1在写object的时候，正在active的T2已经读到了该object的之前版本，这样T1看上去要在T2之后执行，因为T2并没有看到T1更新的数据。定义为T1->T2是RW-反依赖
* 示例 参见paper Figure1 - write skew 和 Figure2 - 三个事务对两张表的batch操作

### 2. 如果以传统方式避免SI不能解决的问题，多需要developer自己改动程序，查找冲突和依赖较为耗时，也很难穷举
现有用SI解决的方案
* 有些负载没有这个问题
* 通过应用程序显示加锁。 SELECT FOR UPDATE tuple级别的锁或者是LOCK TABLE 表级别的锁来阻止其他写
* DBMS自身的实现机制来避免，比如主外键约束等

## SSI的实现
### 主要方向
* PostgreSQL`没有使用S2PL`（两阶段Lock）。S2PL进行操作时上锁，持有直至释放 -> 会带来性能问题
* PostgreSQL`没有使用OCC` （乐观并发控制）。OCC是操作时不上锁，事务提交时检查冲突
* PostgreSQL的SSI是`在SI基础上做的`，在PG9.1里支持。原因是不break已有的用户对SI的认知：读写互相不block，可在快照上直接做

### SSI的三个定理
* T1->T2->T3, T1到T2和T2到T3都是RW反依赖，而且T3最先提交。 T3和T1可以是一个transaction就是write skew问题
* T1和T2有并发，T2和T3有并发
* 只读事务相关的定理：T1->T2->T3反依赖存在，如果T1是只读，而且T3在T1拿快照之前已经提交，这样是有违背seriable隔离级别的。

### SSI怎么存放？
* 存放`依赖关系`：有一个全局的内存数据结构，存放着事务之间的RW反依赖关系。单个事务会记录自己的依赖对象，如果T1->T2->T3这个危险链路存在，就会违背可串行化原则。
* `SIREAD Lock`：单独实现的一个锁，存储读事务读取的对象锁。锁有不同级别：relation级别，page级别，tuple级别。读时会上锁，如果对tuple上锁过多或者其他情况tuple级别锁不满足时（比如gap锁等），可以升级到page，再升级到relation。

### SSI怎么检测？
* 各个事务都`不会block彼此的执行`，读事务在拿`snapshot`时检测冲突，写事务在`commit`时检测冲突。
* 读时冲突检测并记录依赖方法：读一条tuple时，实际进行的是`MVCC`判断，判断tuple的xmin和xmax（tuple里会记录生成自己的transaction id号以及delete自己的transaction id号），对照本事务开始时打快照拿到的活跃事务列表（并发事务），判断tuple对自己是否可见。如果发现该tuple被并发事务所修改，会记录一条RW反依赖记录。 自己->并发事务ID号
* 写时冲突检测并记录依赖方法：写tuple时，会去判断是否有`SIREAD lock`存在，如果有，会记录一条RW反依赖记录。上SIREAD lock的事务ID号->自己

### SSI怎么解决冲突？ （Safe Retry）
* T1->T2->T3 冲突，推迟到T3 commit时解决
* 回滚T2，原因是T2再重试，不会到达原来一样的冲突，因为T3已经commit
* 如果T2和T3都commit了，直接回滚T1，让T1重试

### 只读事务的优化
* Safe Snapshot： 如果只读事务打snapshot时对应的所有的并发事务都提交完毕，可以把这个只读事务的所有SIREAD lock去掉，当作普通的snapshot处理
* Deferrable transaction：大的只读事务的优化。先拿快照，等待active 并发事务完全提交，再做。如果其他事务有RW反依赖，等他们提交后再重新拿一次snapshot，重新等待。实验显示并不会等太久，一般1-6s，不超过20s

### 工程实现优化
* 内存：只读事务优化；锁粒度提升（有可能误杀，但会减少内存使用）；已提交事务SIREADlock及时清理；已提交事务的信息汇总
* 两阶段提交的结合：prepare阶段即将所有事情做好，commit prepare时不回滚
* 主从复制：只允许读节点做safe snapshot读，不考虑读写节点之间的交互图

### 实验结果
* SSI比SI性能慢7%
* SSI比传统的S2PL快很多

## FollowUp
下一次讨论内容：CockRoachDB的SSI的实现. Target Date: 2022.03.22

## Pending Discussion Topics
1. HLC （hybrid-logical clocks）  https://cse.buffalo.edu/tech-reports/2014-04.pdf
2. DDL冲突解决办法。https://research.google.com/pubs/archive/41344.pdf
