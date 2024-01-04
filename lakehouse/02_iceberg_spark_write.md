
## spark引擎中 iceberg 写数据流程
1. spark 引擎层调用某个接口将数据一条一条往下发，iceberg 接受到数据后, 将数据按照指定的格式写入对应的存储中。
2. 在 spark 数据写入完成时，iceberg 按照自己的表规范生成对应的 meta 文件。

## Spark 引擎层和 iceberg 对接

根据 spark 的规范，如果要让 spark 往下游写入数据，需要实现下面几个接口：
1. DataWriter
2. BatchWrite
3. 