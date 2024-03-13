## Hive的几个特点
* Hive最大的特点是通过类SQL来分析大数据，而避免了写MapReduce程序来分析数据，这样使得分析数据更容易。
* 数据是存储在HDFS上的，Hive本身并不提供数据的存储功能
* Hive是将数据映射成数据库和一张张的表，库和表的元数据信息一般存在关系型数据库上（比如MySQL）。
* 数据存储方面：它能够存储很大的数据集，并且对数据完整性、格式要求并不严格。
* 数据处理方面：因为Hive语句最终会生成MapReduce任务去计算，所以不适用于实时计算的场景，它适用于离线分析。
## Hive 架构


![](https://github.com/chenxh/interviews/raw/main/imgs/hive_1.jpg "")

https://github.com/chenxh/interviews/raw/main/imgs/71f12473-b0d9-436b-b0ce-fce60a71a343.png

**Cli命令行**
通过命令行运行 HiveSql。 不通过 HiveServer2。

**HiveServer2**
Hive 通过 HiveServer2 提供 JDBC 协议接口。 HiveServer2 会连接 MetaStoreServer。




## Hive重要概念

### 外部表和内部表
**内部表（managed table）**
* 默认创建的是内部表（managed table），存储位置在hive.metastore.warehouse.dir设置，默认位置是/user/hive/warehouse。
* 导入数据的时候是将文件剪切（移动）到指定位置，即原有路径下文件不再存在
* 删除表的时候，数据和元数据都将被删除
* 默认创建的就是内部表create table xxx (xx xxx)

**外部表（external table）**
* 外部表文件可以在外部系统上，只要有访问权限就可以
* 外部表导入文件时不移动文件，仅仅是添加一个metadata
* 删除外部表时原数据不会被删除
* 分辨外部表内部表可以使用DESCRIBE FORMATTED table_name命令查看
* 创建外部表命令添加一个external即可，即create external table xxx (xxx)
* 外部表指向的数据发生变化的时候会自动更新，不用特殊处理

### 分区表和桶表
**分区（partioned）**
* 有些时候数据是有组织的，比方按日期/类型等分类，而查询数据的时候也经常只关心部分数据，比方说我只想查2017年8月8号，此时可以创建分区，查询具体某一天的数据时，不需要扫描全部目录，所以会明显优化性能
* 一个Hive表在HDFS上是有一个对应的目录来存储数据，普通表的数据直接存储在这个目录下，而分区表数据存储时，是再划分子目录来存储的
* 使用partioned by (xxx)来创建表的分区

**分桶（clustered）**
* 分桶是相对分区进行更细粒度的划分。分桶将整个数据内容安装某列属性值得hash值进行区分，按照取模结果对数据分桶。如取模结果相同的数据记录存放到一个文件。
* 桶表也是一种用于优化查询而设计的表类型。创建通表时，指定桶的个数、分桶的依据字段，hive就可以自动将数据分桶存储。查询时只需要遍历一个桶里的数据，或者遍历部分桶，这样就提高了查询效率。


具体说明分桶

* clustered by (user_id) sorted by(leads_id) into 10 buckets
* clustered by是指根据user_id的值进行哈希后模除分桶个数，根据得到的结果，确定这行数据分入哪个桶中，这样的分法，可以确保相同user_id的数据放入同一个桶中。
* sorted by 是指定桶中的数据以哪个字段进行排序，排序的好处是，在join操作时能获得很高的效率。
* into 10 buckets是指定一共分10个桶。
* 在HDFS上存储时，一个桶存入一个文件中，这样根据user_id进行查询时，可以快速确定数据存在于哪个桶中，而只遍历一个桶可以提供查询效率。

### Hive文件格式

hive文件存储格式包括以下几类：

* TEXTFILE
* SEQUENCEFILE
* RCFILE
* ORCFILE(0.11以后出现)
* parquet