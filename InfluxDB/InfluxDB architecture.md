# 阅读材料
https://docs.influxdata.com/influxdb/v2.2/reference/key-concepts/ 


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
Telegraf agent或者library有transform data的功能，比如转换格式 （MQTT？），网络链路有问题时作backoff等。
