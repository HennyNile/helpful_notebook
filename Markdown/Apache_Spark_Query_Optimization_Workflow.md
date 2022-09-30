# Apache Spark Query Optimization Workflow

 The primary workflow for executing relational queries using Spark is implemented in file **spark\sql\core\src\main\scala\org\apache\spark\sql\execution\QueryExecution.scala**'s class **QueryExecution** . In this workflow, query has been parsed to a logical plan in the beginning and the process of parse is omitted.

## 0. Used files and corresponding file paths

```bash
QueryExecution.scala
	spark\sql\core\src\main\scala\org\apache\spark\sql\execution\QueryExecution.scala
Analyzer.scala
	spark\sql\catalyst\src\main\scala\org\apache\spark\sql\catalyst\analysis\Analyzer.scala
RuleExecutor.scala
	spark\sql\catalyst\src\main\scala\org\apache\spark\sql\catalyst\rules\RuleExecutor.scala
```

## 1. Parsed Logical Plan

```scala
// QueryExecution.scala
class QueryExecution

val logical: LogicalPlan,
```

## 2. Analyzed Logical Plan

```scala
// QueryExecution.scala
class QueryExecution

lazy val analyzed: LogicalPlan = executePhase(QueryPlanningTracker.ANALYSIS) {
  // We can't clone `logical` here, which will reset the `_analyzed` flag.
  sparkSession.sessionState.analyzer.executeAndCheck(logical, tracker)
}
```

```scala
// Analyzer.scala
class Analyzer extends RuleExecutor

def executeAndCheck(plan: LogicalPlan, tracker: QueryPlanningTracker): LogicalPlan = {
  if (plan.analyzed) return plan
  AnalysisHelper.markInAnalyzer {
    val analyzed = executeAndTrack(plan, tracker)
    try {
      checkAnalysis(analyzed)
      analyzed
    } catch {
      case e: AnalysisException =>
        val ae = e.copy(plan = Option(analyzed))
        ae.setStackTrace(e.getStackTrace)
        throw ae
    }
  }
}

// Rule EliminateUnions is shown in the next code block 
override def batches: Seq[Batch] = Seq(
    Batch("Substitution", fixedPoint,
      // This rule optimizes `UpdateFields` expression chains so looks more like optimization rule.
      // However, when manipulating deeply nested schema, `UpdateFields` expression tree could be
      // very complex and make analysis impossible. Thus we need to optimize `UpdateFields` early
      // at the beginning of analysis.
      OptimizeUpdateFields,
      CTESubstitution,
      WindowsSubstitution,
      EliminateUnions,
      SubstituteUnresolvedOrdinals),
    Batch("Disable Hints", Once,
      new ResolveHints.DisableHints),
    Batch("Hints", fixedPoint,
      ResolveHints.ResolveJoinStrategyHints,
      ResolveHints.ResolveCoalesceHints),
    Batch("Simple Sanity Check", Once,
      LookupFunctions),
    Batch("Keep Legacy Outputs", Once,
      KeepLegacyOutputs),
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
    Batch("Remove TempResolvedColumn", Once, RemoveTempResolvedColumn),
    Batch("Apply Char Padding", Once,
      ApplyCharTypePadding),
    Batch("Post-Hoc Resolution", Once,
      Seq(ResolveCommandsWithIfExists) ++
      postHocResolutionRules: _*),
    Batch("Remove Unresolved Hints", Once,
      new ResolveHints.RemoveAllHints),
    Batch("Nondeterministic", Once,
      PullOutNondeterministic),
    Batch("UDF", Once,
      HandleNullInputsForUDF,
      ResolveEncodersInUDF),
    Batch("UpdateNullability", Once,
      UpdateAttributeNullability),
    Batch("Subquery", Once,
      UpdateOuterReferences),
    Batch("Cleanup", fixedPoint,
      CleanupAliases),
    Batch("HandleAnalysisOnlyCommand", Once,
      HandleAnalysisOnlyCommand)
)
```

```scala
// Analyzer.scala

/**
 * Removes [[Union]] operators from the plan if it just has one child.
 */
object EliminateUnions extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsWithPruning(
    _.containsPattern(UNION), ruleId) {
    case u: Union if u.children.size == 1 => u.children.head
  }
}
```

```scala
// RuleExecutor.scala
class RuleExecutor

/** Defines a sequence of rule batches, to be overridden by the implementation. */
protected def batches: Seq[Batch]

def executeAndTrack(plan: TreeType, tracker: QueryPlanningTracker): TreeType = {
  QueryPlanningTracker.withTracker(tracker) {
    execute(plan)
  }
}

/**
 * Executes the batches of rules defined by the subclass. The batches are executed serially
 * using the defined execution strategy. Within each batch, rules are also executed serially.
 */
def execute(plan: TreeType): TreeType = {
  var curPlan = plan
  val queryExecutionMetrics = RuleExecutor.queryExecutionMeter
  val planChangeLogger = new PlanChangeLogger[TreeType]()
  val tracker: Option[QueryPlanningTracker] = QueryPlanningTracker.get
  val beforeMetrics = RuleExecutor.getCurrentMetrics()

  // Run the structural integrity checker against the initial input
  if (!isPlanIntegral(plan, plan)) {
    throw QueryExecutionErrors.structuralIntegrityOfInputPlanIsBrokenInClassError(
      this.getClass.getName.stripSuffix("$"))
  }

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

## 3. Optimized Logical Plan

```scala
// QueryExecution.scala
class QueryExecution

lazy val commandExecuted: LogicalPlan = mode match {
    case CommandExecutionMode.NON_ROOT => analyzed.mapChildren(eagerlyExecuteCommands)
    case CommandExecutionMode.ALL => eagerlyExecuteCommands(analyzed)
    case CommandExecutionMode.SKIP => analyzed
}

lazy val withCachedData: LogicalPlan = sparkSession.withActive {
    assertAnalyzed()
    assertSupported()
    // clone the plan to avoid sharing the plan instance between different stages like analyzing,
    // optimizing and planning.
    sparkSession.sharedState.cacheManager.useCachedData(commandExecuted.clone())
}

lazy val optimizedPlan: LogicalPlan = {
  // We need to materialize the commandExecuted here because optimizedPlan is also tracked under
  // the optimizing phase
  assertCommandExecuted()
  executePhase(QueryPlanningTracker.OPTIMIZATION) {
    // clone the plan to avoid sharing the plan instance between different stages like analyzing,
    // optimizing and planning.
    val plan =
      sparkSession.sessionState.optimizer.executeAndTrack(withCachedData.clone(), tracker)
    // We do not want optimized plans to be re-analyzed as literals that have been constant
    // folded and such can cause issues during analysis. While `clone` should maintain the
    // `analyzed` state of the LogicalPlan, we set the plan as analyzed here as well out of
    // paranoia.
    plan.setAnalyzed()
    plan
  }
}
```

## 4. Execution Plan

```scala
lazy val sparkPlan: SparkPlan = {
  // We need to materialize the optimizedPlan here because sparkPlan is also tracked under
  // the planning phase
  assertOptimized()
  executePhase(QueryPlanningTracker.PLANNING) {
    // Clone the logical plan here, in case the planner rules change the states of the logical
    // plan.
    QueryExecution.createSparkPlan(sparkSession, planner, optimizedPlan.clone())
  }
}
```

```scala
// executedPlan should not be used to initialize any SparkPlan. It should be
// only used for execution.
lazy val executedPlan: SparkPlan = {
  // We need to materialize the optimizedPlan here, before tracking the planning phase, to ensure
  // that the optimization time is not counted as part of the planning phase.
  assertOptimized()
  executePhase(QueryPlanningTracker.PLANNING) {
    // clone the plan to avoid sharing the plan instance between different stages like analyzing,
    // optimizing and planning.
    QueryExecution.prepareForExecution(preparations, sparkPlan.clone())
  }
}
```
