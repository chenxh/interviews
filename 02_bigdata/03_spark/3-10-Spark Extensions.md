## Spark 的扩展机制
Apache Spark是大数据处理领域最常用的计算引擎之一，被应用在各种各样的场景中，除了易用的API，稳定高效的处理引擎，可扩展性也是Spark能够得到广泛应用的一个重要原因。

Spark 的扩展点比较常见包括两个：
1. DataSource V2 扩展，用于支持第三方数据源的读写， Spark 3.0 之后增加了 Catalog 扩展，支持 MultiCatalog 。
2. Spark Catalyst 扩展， 用于自定义语法，自定义执行优化等。

Catalyst 是 Spark 的语法解析引擎，并且能致性优化。 与一般的优化引擎（Apache Calcite等）一样，执行流程包括： 解析SQL 为语法树，转换语法树为逻辑计划，在逻辑计划上实施优化，转化为物理计划。

## Spark Catalyst 扩展点
Spark catalyst的扩展点在SPARK-18127中被引入，Spark用户可以在SQL处理的各个阶段扩展自定义实现。

### SparkSessionExtensions
SparkSessionExtensions保存了所有用户自定义的扩展规则，自定义规则保存在成员变量中，对于不同阶段的自定义规则，SparkSessionExtensions提供了不同的接口。下面是 Iceberg 扩展的部分示例。

```
class IcebergSparkSessionExtensions extends (SparkSessionExtensions => Unit) {
  override def apply(extensions: SparkSessionExtensions): Unit = {
    // parser extensions
    extensions.injectParser { case (_, parser) => new IcebergSparkSqlExtensionsParser(parser) }
    // analyzer extensions
    extensions.injectResolutionRule { spark => ResolveProcedures(spark) }
    extensions.injectCheckRule { _ => MergeIntoIcebergTableResolutionCheck }
        // optimizer extensions
    extensions.injectOptimizerRule { _ => ExtendedSimplifyConditionalsInPredicate }
    extensions.injectPreCBORule { _ => OptimizeMetadataOnlyDeleteFromIcebergTable }
        // planner extensions
    extensions.injectPlannerStrategy { spark => ExtendedDataSourceV2Strategy(spark) } 

。。。
```

新增自定义规则
用户可以通过 SparkSessionExtensions 提供的inject开头的方法添加新的自定义规则，具体的inject接口如下：

* injectParser – 添加parser自定义规则，parser负责SQL解析。
* injectResolutionRule – 添加Analyzer自定义规则到Resolution阶段，analyzer负责逻辑执行计划生成。
* injectOptimizerRule – 添加optimizer自定义规则，optimizer负责逻辑执行计划的优化。
* injectPlannerStrategy – 添加planner strategy自定义规则，planner负责物理执行计划的生成。
* injectPostHocResolutionRule – 添加Analyzer自定义规则到Post Resolution阶段。
* injectCheckRule – 添加Analyzer自定义Check规则。


### 扩展 Parser 示例
通过扩展 Parser , 禁止使用 select * 这种查询所有列的低性能语法。实现步骤如下：
1. 继承 org.apache.spark.sql.catalyst.parser.ParserInterface ，实现 select * 的拦截
```
class CustomSparkSqlExtensionsParser (delegate: ParserInterface) extends ParserInterface {
  override def parsePlan(sqlText: String): LogicalPlan = {
    val logicalPlan = delegate.parsePlan(sqlText)
    logicalPlan transform {
      case project @ Project(projectList, _) =>
        projectList.foreach {
          name =>
            if (name.isInstanceOf[UnresolvedStar]) {  //如果是 select * 的 Project ，抛出异常
              throw new RuntimeException("You must specify your project column set," +
                " * is not allowed.")
            }
        }
        project
    }
    logicalPlan
  }

  override def parseExpression(sqlText: String): Expression = delegate.parseExpression(sqlText)
  override def parseTableIdentifier(sqlText: String): TableIdentifier = delegate.parseTableIdentifier(sqlText)
  override def parseFunctionIdentifier(sqlText: String): FunctionIdentifier = delegate.parseFunctionIdentifier(sqlText)
  override def parseMultipartIdentifier(sqlText: String): Seq[String] = delegate.parseMultipartIdentifier(sqlText)
  override def parseTableSchema(sqlText: String): StructType = delegate.parseTableSchema(sqlText)
  override def parseDataType(sqlText: String): DataType = delegate.parseDataType(sqlText)
  override def parseQuery(sqlText: String): LogicalPlan = delegate.parseQuery(sqlText)
}
```
2. 继承 SparkSessionExtensions  ，注入自定义 Parser 
```
class CustomSparkSessionExtensions extends (SparkSessionExtensions => Unit){
  override def apply(extensions: SparkSessionExtensions): Unit = {
    extensions.injectParser { case (_, parser) => new CustomSparkSqlExtensionsParser(parser) }
  }
}
```
3. 编写测试代码，本地运行
```
public class SparkTest {

    public static void main(String[] args) {
        SparkSession spark = SparkSession.builder()
                .master("local[2]")
                .config("spark.sql.extensions", CustomSparkSessionExtensions.class.getName())
                .getOrCreate();
        spark.sql("select * from test").show();
        spark.close();
    }
}
```
运行结果，应用抛出异常：
```
Exception in thread "main" java.lang.RuntimeException: You must specify your project column set, * is not allowed.
```

### 扩展  optimizer 规则示例
自定义一个 optimizer  规则， 优化 select col * 1 这种查询方式， 因为这种方式有多余的计算 “ * 1”， 直接优化为 select col 形式。
测试代码为：
```
        Dataset df = spark.read().option("header","true").csv("src/main/resources/sales.csv");
        System.out.println(df
                .selectExpr("aa * 1")
                .queryExecution()
                .optimizedPlan()
                .numberedTreeString());
```

***优化前***
优化前的查询计划输出为：
```
00 Project [(cast(aa#17 as double) * 1.0) AS (aa * 1)#21]
01 +- Relation [aa#17,bb#18] csv
```
执行计划包括两个部分：
00 Project - 标示Project投影操作。
01 Relation - 标示我们通过csv文件创建的表。

投影 Project 中 aa * 1 默认不会优化减少计算， 可以定义 optimizer   规则优化。

***自定义优化规则***
继承 org.apache.spark.sql.catalyst.rules.Rule 实现优化
```
object MultiplyOptimizationRule extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan transformAllExpressions {
    // 如果是乘法表达式，并且右边数据等于1， 则转化为左边表达式
    case Multiply(left,right, failOnError) if right.isInstanceOf[Literal] &&
      right.asInstanceOf[Literal].value.asInstanceOf[Double] == 1.0 =>
      println("optimization of one applied")
      left
  }
}
```

在 CustomSparkSessionExtensions 中注入规则
```
class CustomSparkSessionExtensions extends (SparkSessionExtensions => Unit){
  override def apply(extensions: SparkSessionExtensions): Unit = {
    extensions.injectParser { case (_, parser) => new CustomSparkSqlExtensionsParser(parser) }

    //注入自定义优化规则
    extensions.injectOptimizerRule { _ => MultiplyOptimizationRule }
  }
}
```

***优化后***
再次运行测试代码，输出的查询计划为
```
optimization of one applied
00 Project [cast(aa#17 as double) AS (aa * 1)#21]
01 +- Relation [aa#17,bb#18] csv
```

可以看到查询计划中 Project 虽然列名仍然是 aa * 1， 但是计算步骤少了 * 1。

## 总结
本文示例基于 Spark 3.3.1 实现，只测试了解析扩展、执行计划规则扩展。
用户可以在Spark session中自定义自己的parser，analyzer，optimizer以及physical planning stragegy rule。
Iceberg， Hudi 等湖仓平台对接 Spark 引擎，并且定制了语法，数据源和优化等各个环节来实现一个完整湖仓平台。










