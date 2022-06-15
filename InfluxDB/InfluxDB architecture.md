
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


![Measurements & Tags & Fields]（https://github.com/ictmalili/data-ranger/blob/master/InfluxDB/Measurement%20%26%20Tag%20%26%20Fields.png）
* 在Measurement内部，有不同的列，必不可少的是时间列，再有比如Tag列，Field列。
* 可以将Tag列看作是带索引的列，因为查找起来更快。



