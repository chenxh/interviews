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

SQL() 方法返回 DataFrame（Dataset[Row]），DataFrame 包含了 QueryExecution。 

   DataFrame.show() 会触发QueryExecution 计算， 

```
  def head(n: Int): Array[T] = withAction("head", limit(n).queryExecution)(collectFromPlan)
```

collectFromPlan 会调用 sparkPlan 的 execute() 方法。




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
5. optimizedPlan：优化后LogicalPlan。 根据规则优化。
6. sparkPlan：Spark执行计划
7. executedPlan ：物理执行计划。
8. 执行查询 executedPlan， 然后处理数据



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

        // 循环跳出规则： 如果某次规则在生效后，plan 已经不改变了，就退出循环。
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

***Hints 解析***
Hints 机制是在 SQL 语句通过 ```/*+ hint [ , ... ] */``` 语法优化 spark 查询计划的功能。
其中 Join 有下面几种
```
BROADCAST
SHUFFLE_MERGE
SHUFFLE_HASH
SHUFFLE_REPLICATE_NL
```

Join hints 的功能在 ResolveHints.ResolveJoinStrategyHints 中实现。
```
    def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsUpWithPruning(
      _.containsPattern(UNRESOLVED_HINT), ruleId) {
      case h: UnresolvedHint if STRATEGY_HINT_NAMES.contains(h.name.toUpperCase(Locale.ROOT)) =>
        if (h.parameters.isEmpty) {
          // If there is no table alias specified, apply the hint on the entire subtree.
          ResolvedHint(h.child, createHintInfo(h.name))
        } else {
          // Otherwise, find within the subtree query plans to apply the hint.
          val relationNamesInHint = h.parameters.map {
            case tableName: String => UnresolvedAttribute.parseAttributeName(tableName)
            case tableId: UnresolvedAttribute => tableId.nameParts
            case unsupported =>
              throw QueryCompilationErrors.joinStrategyHintParameterNotSupportedError(unsupported)
          }.toSet
          val relationsInHintWithMatch = new mutable.HashSet[Seq[String]]
          val applied = applyJoinStrategyHint(
            h.child, relationNamesInHint, relationsInHintWithMatch, h.name)

          // Filters unmatched relation identifiers in the hint
          val unmatchedIdents = relationNamesInHint -- relationsInHintWithMatch
          hintErrorHandler.hintRelationsNotFound(h.name, h.parameters, unmatchedIdents)
          applied
        }
    }
```
1. plan.resolveOperatorsUpWithPruning 递归调用 rule 应用在子节点上，不适合规则的节点不变。 并返回新的 LogicalPlan。
2. 如果是 hints 节点。开始处理。
3. 如果 hints 节点没有参数， 整个子节点当成一个 table ，构建 ResolvedHint 返回
4. 如果 hints 节点有参数，先解析 tableName，然后执行 applyJoinStrategyHint（）方法，此方法递归子节点查找 tableName 对应 table ，并创建 ResolvedHint 返回。

可以看出，这里只是把 UnresolvedHint 转化为 ResolvedHint。 ResolvedHint 中包含了：1. 具体的 JoinStrategyHint（策略）2. hint 对应的 table 的 LogicalPlan。


**SparkOptimizer 优化 LogicalPlan**

主要组件 SparkOptimizer ， 继承于 Optimizer -> RuleExecutor. 
执行逻辑和 Analyzer 类似。

SparkOptimizer 提供的是逻辑上的优化。例如：
算子下推：把算子下推的子查询，提前计算，提前过滤，加速性能。
算子合并：多个算子合并成一个，减少计算量。
常量折叠：常量表达式直接计算成常量值。


SparkOptimizer 只是基于规则的优化 rule-base。

**SparkPlanner 生成 SparkPlan**

SparkPlanner 会使用一系列的策略来生成一组候选的物理执行计划。

```
  override def strategies: Seq[Strategy] =
    experimentalMethods.extraStrategies ++
      extraPlanningStrategies ++ (
      LogicalQueryStageStrategy ::
      PythonEvals ::
      new DataSourceV2Strategy(session) ::
      FileSourceStrategy ::
      DataSourceStrategy ::
      SpecialLimits ::
      Aggregation ::
      Window ::
      JoinSelection ::
      InMemoryScans ::
      SparkScripts ::
      BasicOperators :: Nil)
```

SparkPlanner.plan
  -> SparkStrategies.plan
    -> QueryPlanner.plan

QueryPlanner.plan 传入 LogicalPlan， 返回 Iterator[PhysicalPlan]。 
```
 // Collect physical plan candidates.
    val candidates = strategies.iterator.flatMap(_(plan))

      // The candidates may contain placeholders marked as [[planLater]],
    // so try to replace them by their child plans.
    val plans = candidates.flatMap { candidate =>
      val placeholders = collectPlaceholders(candidate)

      if (placeholders.isEmpty) {
        // Take the candidate as is because it does not contain placeholders.
        Iterator(candidate)
      } else {
        // Plan the logical plan marked as [[planLater]] and replace the placeholders.
        placeholders.iterator.foldLeft(Iterator(candidate)) {
          case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
            // Plan the logical plan for the placeholder.
            val childPlans = this.plan(logicalPlan)

            candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
              childPlans.map { childPlan =>
                // Replace the placeholder by the child plan
                candidateWithPlaceholders.transformUp {
                  case p if p.eq(placeholder) => childPlan
                }
              }
            }
        }
      }
    }

    val pruned = prunePlans(plans)
    assert(pruned.hasNext, s"No plan for $plan")
    pruned
```


1. 调用 strategies 生成 Iterator[PhysicalPlan]。
2. 替换 PhysicalPlan 中的标记为 planLater 的 LogicalPlan。 上一步中的每个 Strategy 只负责自己关心的 LogicalPlan， 其它的节点会在这一步递归处理。
3. 找到  planLater 的 LogicalPlan， 然后递归 调用 strategies 生成 Iterator[PhysicalPlan]。
   


SparkStrategies.plan

```
  override def plan(plan: LogicalPlan): Iterator[SparkPlan] = {
    super.plan(plan).map { p =>
      val logicalPlan = plan match {
        case ReturnAnswer(rootPlan) => rootPlan
        case _ => plan
      }
      p.setLogicalLink(logicalPlan)
      p
    }
  }
```
设置 LogicalLink 的值。

QueryExecution.createSparkPlan()
```
  def createSparkPlan(
      sparkSession: SparkSession,
      planner: SparkPlanner,
      plan: LogicalPlan): SparkPlan = {
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(plan)).next()
  }
```

***这里只是取了第一个plan， 没有实现 cbo?***


每个 Strategy 的功能就是生成 SparkPlan 。 SparkPlan 也是树形结构，大量的以 exec 结尾的实现就物理计划。
在  SparkPlan 的 doExecute() 中会把 SparkPlan 转化为 RDD 运行。
FilterExec 的 doExecute() 方法
```
  protected override def doExecute(): RDD[InternalRow] = {
    val numOutputRows = longMetric("numOutputRows")
    //子节点运行后生成RDD，再执行过滤。
    child.execute().mapPartitionsWithIndexInternal { (index, iter) =>
      //生成过滤条件
      val predicate = Predicate.create(condition, child.output)
      predicate.initialize(0)
      iter.filter { row =>
        val r = predicate.eval(row)
        if (r) numOutputRows += 1
        r
      }
    }
  }
```


**生成  executedPlan ：物理执行计划**

生成 executedPlan 是调用下面方法实现 
```
 QueryExecution.prepareForExecution(preparations, sparkPlan.clone())
```

此方法主要功能是调用集合 preparations:  Seq[Rule[SparkPlan]] 中规则，修改 SparkPlan 。
preparations 中的规则的主要能力是：
1. 规则 InsertAdaptiveSparkPlan： 作用是在运行时根据运行时数据的统计执行优化运行。 就是适配运行，配置参数```spark.sql.adaptive.enabled=true``` 后生效。
2. 


  






















