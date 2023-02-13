
https://www.dremio.com/resources/guides/apache-iceberg-an-architectural-look-under-the-covers/
## Iceberg
Iceberg 是一个针对巨大、用于分析的表的高性能表格式(Format)。 Iceberg 支持 Spark, Trino, PrestoDB, Flink, Hive and Impala 作为计算引擎，可以像SQL 表一样使用这种高性能表。
本文试图通过分析Iceberg的文件格式和读写流程来理解Iceberg对湖仓要求的支持。
湖仓平台的特点：
1、开放的直接访问数据格式，例如 Apache Parquet 和 ORC，
2、对机器学习和数据科学任务的高效支持
3、一流的性能（SQL）。
4、可靠的数据管理，对事务的支持。

## 什么是表格式？
组织一个表的数据文件、元数据的方式。

## Hive 的表格式
![](https://github.com/chenxh/interviews/blob/main/imgs/hive-table-format-high-level-arch-1.png " ")

**优点**
* 标准透明的表格式， 可以适配很多的处理引擎
* 提供了基础的全表扫描方式，也提供了其它更有效的访问方式：分区，分桶
* 


**缺点**
* 修改数据性能低
* 
