## sparksql


https://zhuanlan.zhihu.com/p/367590611

## 执行流程


val df = sparkSession.sql(sqlText)
df.show()

```
def sql(sqlText: String): DataFrame = withActive {
  // 用于跟踪查询计划的执行，例如：查询计划要执行计划要执行哪些Rule、跟踪记录各个阶段执行的时间等。
  val tracker = new QueryPlanningTracker
  // 调用measurePhase统计解析执行计划的时间。
  // 这是一个高阶函数：def measurePhase[T](phase: String)(f: => T):
  // 执行一个操作，并计算其执行的时间。
  val plan = tracker.measurePhase(QueryPlanningTracker.PARSING) {
    // SessionState的SQL Parser负责解析SQL，并生成解析的执行计划
    // 接口定义为：def parsePlan(sqlText: String): LogicalPlan
    sessionState.sqlParser.parsePlan(sqlText)
  }
  // 生成物理执行计划并生成DataSet（就是DataFrame）
  Dataset.ofRows(self, plan, tracker)
}
```




**LogicPlan**
LogicPlan，逻辑执行计划。从类的继承树可以看出， SQL中所有的节点都有对应的实现类，代表了 sparksql 支持的语法。
父类 TreeNode 。

sessionState.sqlParser.parsePlan(sqlText) 只是将 sql 语句解析为 LogicPlan ， 相当于语法树。

**Dataset.ofRows**

```
 : DataFrame = sparkSession.withActive {
    val qe = new QueryExecution(sparkSession, logicalPlan, tracker)  //创建QueryExecution
    qe.assertAnalyzed() // 解析 logicalPlan 
    new Dataset[Row](qe, RowEncoder(qe.analyzed.schema)) // 创建DataSet，获取解析后的逻辑执行计划对应的schema，这个schema其实就是DataSet要使用的schema。
 }
```

**QueryExecution**
sparksql 执行 SQL 的主要功能类。
传入的 LogicalPlan 的处理顺序为：
1. LogicalPlan ： 根据 sql 解析语法树
2. analyzed LogicalPlan ： 解析后的逻辑执行计划， 使用 [[SessionCatalog]] 中的信息将 [[UnresolvedAttribute]] 和 [[UnresolvedRelation]] 转换为全类型对象。
3. commandExecuted：根据运行mode， 把需要提前运行的操作先执行，例如 DDL 语句。   
4. withCachedData LogicalPlan：添加缓存信息 TODO：
5. optimizedPlan：优化后LogicalPlan。 使用优化器（CBO）优化 。
6. sparkPlan：Spark执行计划
7. executedPlan ：物理执行计划。



**解析生成 LogicPlan**


sessionState.sqlParser 默认为 SparkSqlParser。 

SparkSqlParser.parsePlan
  ParseDriver.parse[T](command: String)(toResult: SqlBaseParser => T)


ParseDriver.parse 使用 antlr 4 来解析语法树， 生成 LogicPlan。
antlr 的 语法定义文件：
sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBaseLexer.g4
sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBaseParser.g4

编译时，运行 antlr 插件生成 SqlBaseLexer和 SqlBaseParser 类。


**生成 analyzed LogicalPlan**

QueryExecution#executePhase
  org.apache.spark.sql.catalyst.analysis.Analyzer#executeAndCheck
    org.apache.spark.sql.catalyst.rules.RuleExecutor#execute


Analyzer 继承 RuleExecutor。execute 方法串行执行子类定义的一些列规则。这些规则由指定的执行策略执行。

```
def execute(plan: TreeType): TreeType = {
    var curPlan = plan
    val queryExecutionMetrics = RuleExecutor.queryExecutionMeter
    val planChangeLogger = new PlanChangeLogger[TreeType]()
    val tracker: Option[QueryPlanningTracker] = QueryPlanningTracker.get
    val beforeMetrics = RuleExecutor.getCurrentMetrics()

    // RuleExecutor中定义方法对初始输入执行初始化检查，确保逻辑执行计划是完整的。 具体逻辑子类实现。
    // 在子类Analyzer中 每个逻辑执行计划都有自己的唯一ID，例如：name#0、cast(name#0 as string) AS name#3。
    // 此处检查需要让逻辑执行计划具备有唯一的ID，且输出表达式ID是否也是唯一的
    // 一旦检测出来有重复的，就会抛出解析失败的异常
    if (!isPlanIntegral(plan, plan)) {
      throw QueryExecutionErrors.structuralIntegrityOfInputPlanIsBrokenInClassError(
        this.getClass.getName.stripSuffix("$"))
    }

    // 串行执行Analyzer组件中定义的每一批解析策略。
    batches.foreach { batch =>
      val batchStartPlan = curPlan
      var iteration = 1
      var lastPlan = curPlan
      var continue = true

      // Run until fix point (or the max number of iterations as specified in the strategy.
      while (continue) {
        curPlan = batch.rules.foldLeft(curPlan) {
          case (plan, rule) =>
            val startTime = System.nanoTime()
            val result = rule(plan)
            val runTime = System.nanoTime() - startTime
            val effective = !result.fastEquals(plan)

            if (effective) {
              queryExecutionMetrics.incNumEffectiveExecution(rule.ruleName)
              queryExecutionMetrics.incTimeEffectiveExecutionBy(rule.ruleName, runTime)
              planChangeLogger.logRule(rule.ruleName, plan, result)
            }
            queryExecutionMetrics.incExecutionTimeBy(rule.ruleName, runTime)
            queryExecutionMetrics.incNumExecution(rule.ruleName)

            // Record timing information using QueryPlanningTracker
            tracker.foreach(_.recordRuleInvocation(rule.ruleName, runTime, effective))

            // Run the structural integrity checker against the plan after each rule.
            if (effective && !isPlanIntegral(plan, result)) {
              throw QueryExecutionErrors.structuralIntegrityIsBrokenAfterApplyingRuleError(
                rule.ruleName, batch.name)
            }

            result
        }
        iteration += 1
        if (iteration > batch.strategy.maxIterations) {
          // Only log if this is a rule that is supposed to run more than once.
          if (iteration != 2) {
            val endingMsg = if (batch.strategy.maxIterationsSetting == null) {
              "."
            } else {
              s", please set '${batch.strategy.maxIterationsSetting}' to a larger value."
            }
            val message = s"Max iterations (${iteration - 1}) reached for batch ${batch.name}" +
              s"$endingMsg"
            if (Utils.isTesting || batch.strategy.errorOnExceed) {
              throw new RuntimeException(message)
            } else {
              logWarning(message)
            }
          }
          // Check idempotence for Once batches.
          if (batch.strategy == Once &&
            Utils.isTesting && !excludedOnceBatches.contains(batch.name)) {
            checkBatchIdempotence(batch, curPlan)
          }
          continue = false
        }

        if (curPlan.fastEquals(lastPlan)) {
          logTrace(
            s"Fixed point reached for batch ${batch.name} after ${iteration - 1} iterations.")
          continue = false
        }
        lastPlan = curPlan
      }

      planChangeLogger.logBatch(batch.name, batchStartPlan, curPlan)
    }
    planChangeLogger.logMetrics(RuleExecutor.getCurrentMetrics() - beforeMetrics)

    curPlan
  }

```

**spark中的Batch**
这个Batch是逻辑计划执行策略组更容易理解一些。里面包含了针对每一类别LogicalPlan的处理类（rule）。每个Batch需要指定其名称、执行策略、以及规则。

```
  protected case class Batch(name: String, strategy: Strategy, rules: Rule[TreeType]*)
```

Analyzer 中定义了很多Batch， 包含了 
Substitution（替换特定语法，例如 with ，window 等）
Disable Hints （禁用 hints）
Hints（解析 hints 语法）
Simple Sanity Check
Resolution（解析）。
。。。。

主要功能 Resolution Batch 中。 

```
 Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions(v1SessionCatalog) ::
      ResolveNamespace(catalogManager) ::
      new ResolveCatalogs(catalogManager) ::
      ResolveUserSpecifiedColumns ::
      ResolveInsertInto ::
      ResolveRelations ::
      ResolvePartitionSpec ::
      ResolveFieldNameAndPosition ::
      AddMetadataColumns ::
      DeduplicateRelations ::
      ResolveReferences ::
      ResolveExpressionsWithNamePlaceholders ::
      ResolveDeserializer ::
      ResolveNewInstance ::
      ResolveUpCast ::
      ResolveGroupingAnalytics ::
      ResolvePivot ::
      ResolveOrdinalInOrderByAndGroupBy ::
      ResolveAggAliasInGroupBy ::
      ResolveMissingReferences ::
      ExtractGenerator ::
      ResolveGenerate ::
      ResolveFunctions ::
      ResolveAliases ::
      ResolveSubquery ::
      ResolveSubqueryColumnAliases ::
      ResolveWindowOrder ::
      ResolveWindowFrame ::
      ResolveNaturalAndUsingJoin ::
      ResolveOutputRelation ::
      ExtractWindowExpressions ::
      GlobalAggregates ::
      ResolveAggregateFunctions ::
      TimeWindowing ::
      SessionWindowing ::
      ResolveInlineTables ::
      ResolveLambdaVariables ::
      ResolveTimeZone ::
      ResolveRandomSeed ::
      ResolveBinaryArithmetic ::
      ResolveUnion ::
      RewriteDeleteFromTable ::
      typeCoercionRules ++
      Seq(ResolveWithCTE) ++
      extendedResolutionRules : _*),
```

**生成物理计划**

















