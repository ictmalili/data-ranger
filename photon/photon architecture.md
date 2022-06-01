
# Paper

[[SIGMOD+2022]+Photon-+A+Fast+Query+Engine+for+Lakehouse+Systems](https://github.com/ictmalili/data-ranger/blob/master/photon/%5BSIGMOD%2B2022%5D%2BPhoton-%2BA%2BFast%2BQuery%2BEngine%2Bfor%2BLakehouse%2BSystems.pdf)

# Photon
目的是为加速数据湖里非结构化数据的处理。采用了 1）列式存储 2）向量化执行引擎 3）与现有的Spark框架无缝集成，包括task model、内存管理等。

Photon就是一个中间的执行器，处理时采用数据向量化处理的过程。

# 设计考量点：
* 向量化执行 vs code-gen
* 行存 vs 列存
* Java vs C++
* Parial Rollout - 从底部开始的执行算子比如Scan适合转成Photon再转化

# 数据结构
![Photon数据结构](https://github.com/ictmalili/data-ranger/blob/master/photon/Photon-Data%20Structure%20.png)


三部分：
1. data，每一列的数据
2. null，标识是否这一行对应的是空值
3. position list。标记那些行是有效的（active）。之所以设计position list是因为数据在内存中进行了相应处理，比如filter，用一个position list可以来标识是否被filter掉，如果不用的话要进行整个数据的拷贝，浪费内存。可以到最后数据需要传出的时候再汇总传出。

# CodeGen vs SIMD
CodeGen是在执行时动态生成一些代码，可以删除掉对实际要访问的数据不需要的片段，从而加速数据处理的速度。有两种方式：一种是手动可以把想实现的expression可以实现按照各种路径写好要执行的代码，在执行时根据实际数据会自动选择相应分支；另外一种是在执行时编译生成代码片段，比如PostgreSQL里嵌入的代码段里有LLVMBuilder调用的代码。
https://github.com/postgres/postgres/blob/7b7ed046cb2ad9f6efac90380757d5977f0f563f/src/backend/jit/llvm/llvmjit_expr.c
https://github.com/postgres/postgres/blob/8ec569479fc28ddd634a13dc100b36352ec3a3c2/src/include/jit/llvmjit.h

另外CodeGen的一种方法有Weld比较popular，2017年的paper，Spark现在就在用Weld来做codegen。
https://cs.stanford.edu/~matei/papers/2017/cidr_weld.pdf

SIMD是可以把数据向量化处理，通过相同指令处理多份数据，好处是实现相对简单。对于一些算子比如filter过滤水平较高。另外也可以支持hashmap的向量化处理的。查看Photon的TPC-H测试结果，也可以看到对T1的性能优化较高。但其实其他的所有query也是有性能提升的，可能效果没有那么明显（但之所以有性能提升也是因为查询里可能有query的过滤）。

论文里把Codegen和SIMD放在对立的角度。那是否有可能两种技术都用呢？？ Questions left。 其实也有可能。

# Photon与Spark查询计划的结合
Photon是执行器，Databricks runtime在执行时会根据需要把数据传到Photon中，Photon进行相应的处理，再把数据通过transfer发回给Spark plan。

![查询计划](https://github.com/ictmalili/data-ranger/blob/master/photon/Photon%20-%20Plan.png)


