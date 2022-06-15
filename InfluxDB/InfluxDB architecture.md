# 阅读材料
https://docs.influxdata.com/influxdb/v2.2/reference/key-concepts/ 
https://docs.influxdata.com/influxdb/v1.8/concepts/storage_engine/
https://blog.fatedier.com/2016/08/05/detailed-in-influxdb-tsm-storage-engine-one/

# InfluxDB Data Model
https://youtu.be/3qTTqsL27lI 

* Buckets -> Databases in Relational databases
* Measurements -> Tables in Relational databases
* Tags -> Indexed table columns
* Fields -> Non-indexed table columns

Measurements & Buckets & Retention setting
![Measurements & Buckets & Retention setting](https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/Measurements%20%26%20Buckets%20.png)

* 每个Measurement对应关系数据库的表，不同的measurement可以根据特征组在一个bucket里.
* 每个bucket可以单独设置自己的retention policy,设置好retention policy后，InfluxDB会自动将过期数据删除.
*  过期数据删除很重要，因为有的数据是有时效性的，或者可以将就的数据保持成较高维度的数据，比如之前是每10s收集一次，对较久的数据可以变成每分钟的统计信息
* InfluxDB支持使用InfluxQL跨bucket的查找，支持pivot，join等方法。

Measurements & Tags & Fields
![Measurements & Tags & Fields](https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/Measurement%20%26%20Tag%20%26%20Fields.png)
* 在Measurement内部，有不同的列，必不可少的是时间列，再有比如Tag列，Field列。
* 可以将Tag列看作是带索引的列，因为查找起来更快。

# InfluxDB 设计理念
https://docs.influxdata.com/influxdb/v2.2/reference/key-concepts/design-principles/
* Update/Delete被限制。 因为时序数据很少被Update，所以没有conflict update的支持（？） 只能delete旧数据（没有并发的写入操作的数据）
* Read/Write 按最终一致性处理，所以有时读不到刚流进来的数据
* Schemaless design 无需实现定义schema，只需要定义这个measurement名字即可（表名）
* Point 单点的数据没有ID号，直接用时间标识，因为InfluxDB更多是对数据做聚合处理
* 重复的数据 采用直接覆盖的方式。认为相同时间点的数据发过来就是冗余数据

# InfluxDB 数据写入
![Ingest Data to InfluxDB](https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/InfluxDB-Ingest%20Data.png)
支持device直接写或者是通过Gateway写入。也支持云上机器写入
提供REST API，有library支持12种编程语言
也可以通过Telegraf agent来进行处理，Telegraf agent可以部署在device上，也可以部署在gateway上
Telegraf agent或者library有transform data的功能，比如转换格式 （MQTT:The Standard for IoT Messaging），网络链路有问题时作backoff等。

# InfluxDB 数据查询
有两种方式Flux和InfluxQL
Flux 查询有两种：
1. basic 查询，比如指定data source、time range和data filters。 
2. Flux functions。定义了系列方法，比如mean(),window(),duplicate(),aggregateWindows, group, sort, limit, map, histograms, fills(填充NULL值）。常用的一些聚合类操作。

InfluxQL是基于InfluxDB 1.x的model的。现在InfluxDB最新是2.2版本。版本2和1的区别在于版本1的database&retention policy对应版本2里的buckets（关系数据库的database）。

# InfluxDB 底层数据模型 TSM
https://youtu.be/C5sv0CtuMCw
面向用户的模型和TSM模型的对比（没有Tag的情况）
![TSM 2 series:dimension & field](https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/InfluxDB-Internal%20TSM%20-%202%20Series(measurement%2Bfield).png)

底层存储在TSM里存储是以列为单位按series来进行存储的，每个series的key是measurement+field，存储的内容是按列存储的，存放时间和值，值按时间排序。
上图例子里有2个series，就是measurement+对应的field

面向用户的模型和TSM模型的对比（有Tag的情况）
![TSM 4 series:dimension & field & tag](https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/InfluxDB%20-%20Internal%20TSM%20-%204%20series%20(measurement%2Btag%2Bfield).png)

如果有Tag标识，会将每个Tag的值和field作为单独series进行存放
上面例子中Tag是location，location一共有两个可取值，所以series数目乘以2。为4个series

如果Tag设计不好，每个Tag都是单一值，存储series可能会很多 （Question：对于主键经常查询的情况是否需要存成Tag呢？那样就会有非常多的series了）

# InfluxDB的存储引擎
存储数据分为几步：
1. 写WAL Log，fsync到磁盘
2. 数据更新写到Cache, 返回给用户
3. 数据同步到disk。
4. <异步> 同步到disk以后，WAL Log可以truncate，cache里数据可以删除 （Question：一定会删除吗？） 

# TSM vs LSM
TSM比LSM好的地方在于它提出了shard的概念，根据时间将数据分成不同的分片，这样在进行批量delete过期数据的时候性能会好，因为只需要删除相应的chunk即可。反观LSM，它在删除数据时需要先做delete marker，再compact数据，相对复杂。
另外，InfluxDB还提到LSM有文件句柄过多的问题，为什么TSM能解决？ 也许是TSM层数比较少（4 vs 6），也有可能数据在shard中打散，需要打开的数据不需要向后倒更深层时的tree，但TSM也还是存在文件数目过多的问题。

# InfluxDB vs TimeScale/TDEngine
TimeScale是基于PostgreSQL直接构建的，上面通过加额外的一层HyperTable做相应的处理，InfluxDB重写的存储层。关于是否有必要重写存储系统，其实也要看是否在开源上做能达到需求，如果能打到，也是可以的。TimeScale说是在更大规模的数据上（比如更多的传感器），会有更好的性能，所以是否重写引擎就一定快也是不好说的。

# 数据如何打到时序数据库
对于中小规模的数据，可以将设备通过client端或者gateway打过来，当然client端或是gateway可以有batch处理、backoff策略等处理来减轻对数据库的压力。如果对于较大的数据，可以考虑中间加入一层中间件比如Kafka来进行消峰的操作，降低对数据库的影响。

# 时序数据库和实时流处理一样吗？
时序数据库是要求数据进行持久化存储的，实时流处理可能拿到的是更小的时间窗口，或者是它有实时更新的物化视图，当数据流过时，视图自动变化。时序数据库对于数据的时效性可能要求并不是那么高，但存储的数据可能会更久一些。

# 时序数据库一定快于关系数据库上存储时序数据吗？
时序数据库更多是做数据的聚集分析，封装了自己的很多方法，基于时间维度的aggregation分析等，如果关系数据库上来实现，会相对复杂，所以从易用性的角度，时序数据库更强一些。
至于时序数据库底层的实现，通过关系数据库来做也未尝不可。可能对于有些大宽表的处理，要做额外处理，比如PostgreSQL列的上限是1600.

# 下次讨论话题
流数据处理 - Flink
