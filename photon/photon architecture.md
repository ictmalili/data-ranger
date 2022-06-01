
# Paper

[[SIGMOD+2022]+Photon-+A+Fast+Query+Engine+for+Lakehouse+Systems](https://github.com/ictmalili/data-ranger/blob/master/photon/%5BSIGMOD%2B2022%5D%2BPhoton-%2BA%2BFast%2BQuery%2BEngine%2Bfor%2BLakehouse%2BSystems.pdf)

# Photon
目的是为加速数据湖里非结构化数据的处理。采用了 1）列式存储 2）向量化执行引擎 3）与现有的Spark框架无缝集成，包括task model、内存管理等。

Photon就是一个中间的执行器，处理时采用数据向量化处理的过程。

# 设计考量点：
向量化执行 vs code-gen
行存 vs 列存
Java vs C++
Parial Rollout - 从底部开始的执行算子比如Scan适合转成Photon再转化

# 数据结构
三部分：
1. data，每一列的数据
2. null，标识是否这一行对应的是空值
3. position list。标记那些行是有效的。 active
