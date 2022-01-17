# Papers
1. [Socrates: The New SQL Server in the Cloud SIGMOD2019](https://www.microsoft.com/en-us/research/uploads/prod/2019/05/socrates.pdf)



# 1月12日讨论总结 (by lili)

## Socrates和Aurora的区别：

1. Log 单独存放，更加灵活，更好scale，能用更贵硬件
2. 计算层使用了SSD，有Resilient Buffer Pool Extension，能够有更大本地cache空间

## PostgreSQL和MySQL区别

1. PostgreSQL没有undo，删除数据是加标记，所以不用像mysql一样做undo
2. PostgreSQL的每行有xmin，xmax，可以根据数据或者log直接构建出活跃事务列表，aurora mysql其实也可以根据底层log构建，但为方便起见，从主库传输过来了。

## Left questions：

1. Aurora 读节点的读取和共享存储之间的同步。如何保证不读没有flush的页，如何保证buffer pool里刷出去的页后续能够被读到。对照Socrates 4.4节。

2. PolarDB，Socrates，Aurora读节点的事务隔离行。 RR和Snapshot的区别。 -- RR有幻读，Snapshot会有write-skew问题。
