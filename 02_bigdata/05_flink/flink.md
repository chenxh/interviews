## watermark
**问题**
在实际的流式计算中数据到来的顺序对计算结果的正确性有至关重要的影响，比如：某数据源中的某些数据由于某种原因(如：网络原因，外部存储自身原因)会有5秒的延时，也就是在实际时间的第1秒产生的数据有可能在第5秒中产生的数据之后到来(比如到Window处理节点).选具体某个delay的元素来说，假设在一个5秒的Tumble窗口，有一个EventTime是 11秒的数据，在第16秒时候到来了。

**Apache Flink的时间类型**

**ProcessingTime**

是数据流入到具体某个算子时候相应的系统时间。ProcessingTime 有最好的性能和最低的延迟。但在分布式计算环境中ProcessingTime具有不确定性，相同数据流多次运行有可能产生不同的计算结果。

**IngestionTime**

IngestionTime是数据进入Apache Flink框架的时间，是在Source Operator中设置的。与ProcessingTime相比可以提供更可预测的结果，因为IngestionTime的时间戳比较稳定(在源处只记录一次)，同一数据在流经不同窗口操作时将使用相同的时间戳，而对于ProcessingTime同一数据在流经不同窗口算子会有不同的处理时间戳。

**EventTime**

EventTime是事件在设备上产生时候携带的。在进入Apache Flink框架之前EventTime通常要嵌入到记录中，并且EventTime也可以从记录中提取出来。在实际的网上购物订单等业务场景中，大多会使用EventTime来进行数据计算。

**什么是Watermark**
Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Apache Flink Source或者自定义的Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。 Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。 Watermark的产生和Apache Flink内部处理逻辑如下图所示:

![](https://github.com/chenxh/interviews/raw/main/bigdata/imgs/flink_watermark.png "")