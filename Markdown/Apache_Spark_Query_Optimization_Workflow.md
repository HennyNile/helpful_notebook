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
Optimizer.scala
	spark\sql\catalyst\src\main\scala\org\apache\spark\sql\catalyst\optimizer\Optimizer.scala
SparkStrategies.scala
	spark\sql\core\src\main\scala\org\apache\spark\sql\execution\SparkStrategies.scala
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

override def execute(plan: LogicalPlan): LogicalPlan = {
   AnalysisContext.withNewAnalysisContext {
   executeSameContext(plan)
   }
}

private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
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

```scala
// Analyzer.scala

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

Rule of Cost-based Reorder **CostBasedJoinReorder**  is involved in the following batches.

```scala
// Optimizer.scala
class Optimizer extends RuleExecutor

final override def batches: Seq[Batch] = {
    val excludedRulesConf =
    conf.optimizerExcludedRules.toSeq.flatMap(Utils.stringToSeq)
    val excludedRules = excludedRulesConf.filter { ruleName =>
        val nonExcludable = nonExcludableRules.contains(ruleName)
        if (nonExcludable) {
            logWarning(s"Optimization rule '${ruleName}' was not excluded from the optimizer " +
                       s"because this rule is a non-excludable rule.")
        }
        !nonExcludable
    }
    if (excludedRules.isEmpty) {
        defaultBatches
    } else {
        defaultBatches.flatMap { batch =>
            val filteredRules = batch.rules.filter { rule =>
                val exclude = excludedRules.contains(rule.ruleName)
                if (exclude) {
                    logInfo(s"Optimization rule '${rule.ruleName}' is excluded from the optimizer.")
                }
                !exclude
            }
            if (batch.rules == filteredRules) {
                Some(batch)
            } else if (filteredRules.nonEmpty) {
                Some(Batch(batch.name, batch.strategy, filteredRules: _*))
            } else {
                logInfo(s"Optimization batch '${batch.name}' is excluded from the optimizer " +
                        s"as all enclosed rules have been excluded.")
                None
            }
        }
    }
}

def defaultBatches: Seq[Batch] = {
    val operatorOptimizationRuleSet =
    Seq(
        // Operator push down
        PushProjectionThroughUnion,
        PushProjectionThroughLimit,
        ReorderJoin,
        EliminateOuterJoin,
        PushDownPredicates,
        PushDownLeftSemiAntiJoin,
        PushLeftSemiLeftAntiThroughJoin,
        LimitPushDown,
        LimitPushDownThroughWindow,
        ColumnPruning,
        GenerateOptimization,
        // Operator combine
        CollapseRepartition,
        CollapseProject,
        OptimizeWindowFunctions,
        CollapseWindow,
        CombineFilters,
        EliminateOffsets,
        EliminateLimits,
        CombineUnions,
        // Constant folding and strength reduction
        OptimizeRepartition,
        TransposeWindow,
        NullPropagation,
        NullDownPropagation,
        ConstantPropagation,
        FoldablePropagation,
        OptimizeIn,
        OptimizeRand,
        ConstantFolding,
        EliminateAggregateFilter,
        ReorderAssociativeOperator,
        LikeSimplification,
        BooleanSimplification,
        SimplifyConditionals,
        PushFoldableIntoBranches,
        RemoveDispensableExpressions,
        SimplifyBinaryComparison,
        ReplaceNullWithFalseInPredicate,
        PruneFilters,
        SimplifyCasts,
        SimplifyCaseConversionExpressions,
        RewriteCorrelatedScalarSubquery,
        RewriteLateralSubquery,
        EliminateSerialization,
        RemoveRedundantAliases,
        RemoveRedundantAggregates,
        UnwrapCastInBinaryComparison,
        RemoveNoopOperators,
        OptimizeUpdateFields,
        SimplifyExtractValueOps,
        OptimizeCsvJsonExprs,
        CombineConcats,
        PushdownPredicatesAndPruneColumnsForCTEDef) ++
    extendedOperatorOptimizationRules

    val operatorOptimizationBatch: Seq[Batch] = {
        Batch("Operator Optimization before Inferring Filters", fixedPoint,
              operatorOptimizationRuleSet: _*) ::
        Batch("Infer Filters", Once,
              InferFiltersFromGenerate,
              InferFiltersFromConstraints) ::
        Batch("Operator Optimization after Inferring Filters", fixedPoint,
              operatorOptimizationRuleSet: _*) ::
        Batch("Push extra predicate through join", fixedPoint,
              PushExtraPredicateThroughJoin,
              PushDownPredicates) :: Nil
    }

    val batches = (
        Batch("Finish Analysis", Once, FinishAnalysis) ::
        //////////////////////////////////////////////////////////////////////////////////////////
        // Optimizer rules start here
        //////////////////////////////////////////////////////////////////////////////////////////
        Batch("Eliminate Distinct", Once, EliminateDistinct) ::
        // - Do the first call of CombineUnions before starting the major Optimizer rules,
        //   since it can reduce the number of iteration and the other rules could add/move
        //   extra operators between two adjacent Union operators.
        // - Call CombineUnions again in Batch("Operator Optimizations"),
        //   since the other rules might make two separate Unions operators adjacent.
        Batch("Inline CTE", Once,
              InlineCTE()) ::
        Batch("Union", Once,
              RemoveNoopOperators,
              CombineUnions,
              RemoveNoopUnion) ::
        // Run this once earlier. This might simplify the plan and reduce cost of optimizer.
        // For example, a query such as Filter(LocalRelation) would go through all the heavy
        // optimizer rules that are triggered when there is a filter
        // (e.g. InferFiltersFromConstraints). If we run this batch earlier, the query becomes just
        // LocalRelation and does not trigger many rules.
        Batch("LocalRelation early", fixedPoint,
              ConvertToLocalRelation,
              PropagateEmptyRelation,
              // PropagateEmptyRelation can change the nullability of an attribute from nullable to
              // non-nullable when an empty relation child of a Union is removed
              UpdateAttributeNullability) ::
        Batch("Pullup Correlated Expressions", Once,
              OptimizeOneRowRelationSubquery,
              PullupCorrelatedPredicates) ::
        // Subquery batch applies the optimizer rules recursively. Therefore, it makes no sense
        // to enforce idempotence on it and we change this batch from Once to FixedPoint(1).
        Batch("Subquery", FixedPoint(1),
              OptimizeSubqueries) ::
        Batch("Replace Operators", fixedPoint,
              RewriteExceptAll,
              RewriteIntersectAll,
              ReplaceIntersectWithSemiJoin,
              ReplaceExceptWithFilter,
              ReplaceExceptWithAntiJoin,
              ReplaceDistinctWithAggregate,
              ReplaceDeduplicateWithAggregate) ::
        Batch("Aggregate", fixedPoint,
              RemoveLiteralFromGroupExpressions,
              RemoveRepetitionFromGroupExpressions) :: Nil ++
        operatorOptimizationBatch) :+
    Batch("Clean Up Temporary CTE Info", Once, CleanUpTempCTEInfo) :+
    // This batch rewrites plans after the operator optimization and
    // before any batches that depend on stats.
    Batch("Pre CBO Rules", Once, preCBORules: _*) :+
    // This batch pushes filters and projections into scan nodes. Before this batch, the logical
    // plan may contain nodes that do not report stats. Anything that uses stats must run after
    // this batch.
    Batch("Early Filter and Projection Push-Down", Once, earlyScanPushDownRules: _*) :+
    Batch("Update CTE Relation Stats", Once, UpdateCTERelationStats) :+
    // Since join costs in AQP can change between multiple runs, there is no reason that we have an
    // idempotence enforcement on this batch. We thus make it FixedPoint(1) instead of Once.
    Batch("Join Reorder", FixedPoint(1),
          CostBasedJoinReorder) :+
    Batch("Eliminate Sorts", Once,
          EliminateSorts) :+
    Batch("Decimal Optimizations", fixedPoint,
          DecimalAggregates) :+
    // This batch must run after "Decimal Optimizations", as that one may change the
    // aggregate distinct column
    Batch("Distinct Aggregate Rewrite", Once,
          RewriteDistinctAggregates) :+
    Batch("Object Expressions Optimization", fixedPoint,
          EliminateMapObjects,
          CombineTypedFilters,
          ObjectSerializerPruning,
          ReassignLambdaVariableID) :+
    Batch("LocalRelation", fixedPoint,
          ConvertToLocalRelation,
          PropagateEmptyRelation,
          // PropagateEmptyRelation can change the nullability of an attribute from nullable to
          // non-nullable when an empty relation child of a Union is removed
          UpdateAttributeNullability) :+
    Batch("Optimize One Row Plan", fixedPoint, OptimizeOneRowPlan) :+
    // The following batch should be executed after batch "Join Reorder" and "LocalRelation".
    Batch("Check Cartesian Products", Once,
          CheckCartesianProducts) :+
    Batch("RewriteSubquery", Once,
          RewritePredicateSubquery,
          PushPredicateThroughJoin,
          LimitPushDown,
          ColumnPruning,
          CollapseProject,
          RemoveRedundantAliases,
          RemoveNoopOperators) :+
    // This batch must be executed after the `RewriteSubquery` batch, which creates joins.
    Batch("NormalizeFloatingNumbers", Once, NormalizeFloatingNumbers) :+
    Batch("ReplaceUpdateFieldsExpression", Once, ReplaceUpdateFieldsExpression)

    // remove any batches with no rules. this may happen when subclasses do not add optional rules.
    batches.filter(_.rules.nonEmpty)
}
```

The following  is the rule Reorder in the batches of Optimization rules.

```scala
object ReorderJoin extends Rule[LogicalPlan] with PredicateHelper {
    /**
   * Join a list of plans together and push down the conditions into them.
   *
   * The joined plan are picked from left to right, prefer those has at least one join condition.
   *
   * @param input a list of LogicalPlans to inner join and the type of inner join.
   * @param conditions a list of condition for join.
   */
    @tailrec
    final def createOrderedJoin(
        input: Seq[(LogicalPlan, InnerLike)],
        conditions: Seq[Expression]): LogicalPlan = {
        assert(input.size >= 2)
        if (input.size == 2) {
            val (joinConditions, others) = conditions.partition(canEvaluateWithinJoin)
            val ((left, leftJoinType), (right, rightJoinType)) = (input(0), input(1))
            val innerJoinType = (leftJoinType, rightJoinType) match {
                case (Inner, Inner) => Inner
                case (_, _) => Cross
            }
            val join = Join(left, right, innerJoinType,
                            joinConditions.reduceLeftOption(And), JoinHint.NONE)
            if (others.nonEmpty) {
                Filter(others.reduceLeft(And), join)
            } else {
                join
            }
        } else {
            val (left, _) :: rest = input.toList
            // find out the first join that have at least one join condition
            val conditionalJoin = rest.find { planJoinPair =>
                val plan = planJoinPair._1
                val refs = left.outputSet ++ plan.outputSet
                conditions
                .filterNot(l => l.references.nonEmpty && canEvaluate(l, left))
                .filterNot(r => r.references.nonEmpty && canEvaluate(r, plan))
                .exists(_.references.subsetOf(refs))
            }
            // pick the next one if no condition left
            val (right, innerJoinType) = conditionalJoin.getOrElse(rest.head)

            val joinedRefs = left.outputSet ++ right.outputSet
            val (joinConditions, others) = conditions.partition(
                e => e.references.subsetOf(joinedRefs) && canEvaluateWithinJoin(e))
            val joined = Join(left, right, innerJoinType,
                              joinConditions.reduceLeftOption(And), JoinHint.NONE)

            // should not have reference to same logical plan
            createOrderedJoin(Seq((joined, Inner)) ++ rest.filterNot(_._1 eq right), others)
        }
    }

    def apply(plan: LogicalPlan): LogicalPlan = plan.transformWithPruning(
        _.containsPattern(INNER_LIKE_JOIN), ruleId) {
        case p @ ExtractFiltersAndInnerJoins(input, conditions)
        if input.size > 2 && conditions.nonEmpty =>
        val reordered = if (conf.starSchemaDetection && !conf.cboEnabled) {
            val starJoinPlan = StarSchemaDetection.reorderStarJoins(input, conditions)
            if (starJoinPlan.nonEmpty) {
                val rest = input.filterNot(starJoinPlan.contains(_))
                createOrderedJoin(starJoinPlan ++ rest, conditions)
            } else {
                createOrderedJoin(input, conditions)
            }
        } else {
            createOrderedJoin(input, conditions)
        }

        if (p.sameOutput(reordered)) {
            reordered
        } else {
            // Reordering the joins have changed the order of the columns.
            // Inject a projection to make sure we restore to the expected ordering.
            Project(p.output, reordered)
        }
    }
}
```

## 4. Execution Plan

```scala
// QueryExecution.scala
class QueryExecution

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

/**
   * Transform a [[LogicalPlan]] into a [[SparkPlan]].
   *
   * Note that the returned physical plan still needs to be prepared for execution.
   */
def createSparkPlan(
    sparkSession: SparkSession,
    planner: SparkPlanner,
    plan: LogicalPlan): SparkPlan = {
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(plan)).next()
}
```

```scala
// SparkStrategies.scala
class SparkStrategies extends QueryPlanner[SparkPlan]

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

