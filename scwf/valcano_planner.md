# planner 设计

```
abstract class QueryPlanner[PhysicalPlan <: TreeNode[PhysicalPlan]] {
  /** A list of execution strategies that can be used by the planner */
  def strategies: Seq[GenericStrategy[PhysicalPlan]]

  def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
    // Obviously a lot to do here still...

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
                  case p if p == placeholder => childPlan
                }
              }
            }
        }
      }
    }

    val pruned = prunePlans(plans)
    assert(pruned.hasNext, s"No plan for $plan")
    pruned
  }
```



1. from top to down, first plan the root node and added planlater on the top of the children
2. plan chilren and replace the planlater with the childre pysical plans
3. recursively 1 and 2 until touch the leaf node



logical plan  — (rule based optimizer) —> logical plan — (cost base optimizer: Join reorder) —> logical plan 



logical plan —(transformation rules)—> equal logical plans — (strategies) —> physical plans —— > best physical 



calcite 的实现：

```
impatient 是否耐烦，
```

```
TableEnvironment 中有使用calcite的初始化构建过程

// the configuration to create a Calcite planner
private lazy val frameworkConfig: FrameworkConfig = Frameworks
  .newConfigBuilder
  .defaultSchema(tables)
  .parserConfig(getSqlParserConfig)
  .costFactory(new DataSetCostFactory)
  .typeSystem(new FlinkTypeSystem)
  .operatorTable(getSqlOperatorTable)
  // set the executor to evaluate constant expressions
  .executor(new ExpressionReducer(config))
  .build

// the builder for Calcite RelNodes, Calcite's representation of a relational expression tree.
protected lazy val relBuilder: FlinkRelBuilder = FlinkRelBuilder.create(frameworkConfig)

// the planner instance used to optimize queries of this TableEnvironment
private lazy val planner: RelOptPlanner = relBuilder.getPlanner
```



table api，BatchTableEnvironment的 sql方法

```
/**
  * Evaluates a SQL query on registered tables and retrieves the result as a [[Table]].
  *
  * All tables referenced by the query must be registered in the TableEnvironment.
  *
  * @param query The SQL query to evaluate.
  * @return The result of the query as Table.
  */
override def sql(query: String): Table = {

  val planner = new FlinkPlannerImpl(getFrameworkConfig, getPlanner, getTypeFactory)
  // parse the sql query
  val parsed = planner.parse(query)
  // validate the sql query
  val validated = planner.validate(parsed)
  // transform to a relational tree
  val relational = planner.rel(validated)

  new Table(this, LogicalRelNode(relational.rel))
}
```

flink 的优化器实现

```
BatchTableEnvironment 的 optimize 方法
```

```
/**
  * Generates the optimized [[RelNode]] tree from the original relational node tree.
  *
  * @param relNode The original [[RelNode]] tree
  * @return The optimized [[RelNode]] tree
  */
private[flink] def optimize(relNode: RelNode): RelNode = {

  // decorrelate
  val decorPlan = RelDecorrelator.decorrelateQuery(relNode)

  // optimize the logical Flink plan
  // wf: generate RuleSetProgram
  val optProgram = Programs.ofRules(getRuleSet)
  val flinkOutputProps = relNode.getTraitSet.replace(DataSetConvention.INSTANCE).simplify()

  val dataSetPlan = try {
    optProgram.run(getPlanner, decorPlan, flinkOutputProps)
  } catch {
    case e: CannotPlanException =>
      throw new TableException(
        s"Cannot generate a valid execution plan for the given query: \n\n" +
          s"${RelOptUtil.toString(relNode)}\n" +
          s"This exception indicates that the query uses an unsupported SQL feature.\n" +
          s"Please check the documentation for the set of currently supported SQL features.")
    case t: TableException =>
      throw new TableException(
        s"Cannot generate a valid execution plan for the given query: \n\n" +
          s"${RelOptUtil.toString(relNode)}\n" +
          s"${t.msg}\n" +
          s"Please check the documentation for the set of currently supported SQL features.")
    case a: AssertionError =>
      throw a.getCause
  }
  dataSetPlan
}
```

table 类似 dataframe

```
/**
  * A Table is the core component of the Table API.
  * Similar to how the batch and streaming APIs have DataSet and DataStream,
  * the Table API is built around [[Table]].
  *
  * Use the methods of [[Table]] to transform data. Use [[TableEnvironment]] to convert a [[Table]]
  * back to a DataSet or DataStream.
  *
  * When using Scala a [[Table]] can also be converted using implicit conversions.
  *
  * Example:
  *
  * {{{
  *   val env = ExecutionEnvironment.getExecutionEnvironment
  *   val tEnv = TableEnvironment.getTableEnvironment(env)
  *
  *   val set: DataSet[(String, Int)] = ...
  *   val table = set.toTable(tEnv, 'a, 'b)
  *   ...
  *   val table2 = ...
  *   val set2: DataSet[MyType] = table2.toDataSet[MyType]
  * }}}
  *
  * Operations such as [[join]], [[select]], [[where]] and [[groupBy]] either take arguments
  * in a Scala DSL or as an expression String. Please refer to the documentation for the expression
  * syntax.
  *
  * @param tableEnv The [[TableEnvironment]] to which the table is bound.
  * @param logicalPlan logical representation
  */
class Table(
    private[flink] val tableEnv: TableEnvironment,
    private[flink] val logicalPlan: LogicalNode) {
```



```
Questions：
1. FlinkRuleSets里面的rule，是如何执行的，有没有spark里面的batch和收敛的概念？
2. 如何理解一个RelOptRule onMatch， matchs
3. RelOptRuleOperand -- matchs
/**
 * Program that transforms a relational expression into another relational
 * expression.
 *
 * <p>A planner is a sequence of programs, each of which is sometimes called
 * a "phase".
 * The most typical program is an invocation of the volcano planner with a
 * particular {@link org.apache.calcite.tools.RuleSet}.</p>
 */
public interface Program {
  RelNode run(RelOptPlanner planner, RelNode rel,
      RelTraitSet requiredOutputTraits);
} 和 planner的区别

生成的是RuleSetProgram
/** Program backed by a {@link RuleSet}. */
  static class RuleSetProgram implements Program {
    final RuleSet ruleSet;

    private RuleSetProgram(RuleSet ruleSet) {
      this.ruleSet = ruleSet;
    }

    public RelNode run(RelOptPlanner planner, RelNode rel,
        RelTraitSet requiredOutputTraits) {
      planner.clear();
      for (RelOptRule rule : ruleSet) {
        planner.addRule(rule);
      }
      if (!rel.getTraitSet().equals(requiredOutputTraits)) {
        rel = planner.changeTraits(rel, requiredOutputTraits);
      }
      planner.setRoot(rel);
      return planner.findBestExp();

    }
  }
```





```
calcite 的volcano planner 主流程

/**
 * Finds the most efficient expression to implement the query given via
 * {@link org.apache.calcite.plan.RelOptPlanner#setRoot(org.apache.calcite.rel.RelNode)}.
 *
 * <p>The algorithm executes repeatedly in a series of phases. In each phase
 * the exact rules that may be fired varies. The mapping of phases to rule
 * sets is maintained in the {@link #ruleQueue}.
 *
 * <p>In each phase, the planner sets the initial importance of the existing
 * RelSubSets ({@link #setInitialImportance()}). The planner then iterates
 * over the rule matches presented by the rule queue until:
 *
 * <ol>
 * <li>The rule queue becomes empty.</li>
 * <li>For ambitious planners: No improvements to the plan have been made
 * recently (specifically within a number of iterations that is 10% of the
 * number of iterations necessary to first reach an implementable plan or 25
 * iterations whichever is larger).</li>
 * <li>For non-ambitious planners: When an implementable plan is found.</li>
 * </ol>
 *
 * <p>Furthermore, after every 10 iterations without an implementable plan,
 * RelSubSets that contain only logical RelNodes are given an importance
 * boost via {@link #injectImportanceBoost()}. Once an implementable plan is
 * found, the artificially raised importance values are cleared (see
 * {@link #clearImportanceBoost()}).
 *
 * @return the most efficient RelNode tree found for implementing the given
 * query
 */
public RelNode findBestExp() {
  ensureRootConverters();
  useApplicableMaterializations();
  int cumulativeTicks = 0;
  for (VolcanoPlannerPhase phase : VolcanoPlannerPhase.values()) {
    setInitialImportance();

    RelOptCost targetCost = costFactory.makeHugeCost();
    int tick = 0;
    int firstFiniteTick = -1;
    int splitCount = 0;
    int giveUpTick = Integer.MAX_VALUE;

    while (true) {
      ++tick;
      ++cumulativeTicks;
      if (root.bestCost.isLe(targetCost)) {
        if (firstFiniteTick < 0) {
          firstFiniteTick = cumulativeTicks;

          clearImportanceBoost();
        }
        if (ambitious) {
          // Choose a slightly more ambitious target cost, and
          // try again. If it took us 1000 iterations to find our
          // first finite plan, give ourselves another 100
          // iterations to reduce the cost by 10%.
          targetCost = root.bestCost.multiplyBy(0.9);
          ++splitCount;
          if (impatient) {
            if (firstFiniteTick < 10) {
              // It's possible pre-processing can create
              // an implementable plan -- give us some time
              // to actually optimize it.
              giveUpTick = cumulativeTicks + 25;
            } else {
              giveUpTick =
                  cumulativeTicks
                      + Math.max(firstFiniteTick / 10, 25);
            }
          }
        } else {
          break;
        }
      } else if (cumulativeTicks > giveUpTick) {
        // We haven't made progress recently. Take the current best.
        break;
      } else if (root.bestCost.isInfinite() && ((tick % 10) == 0)) {
        injectImportanceBoost();
      }

      LOGGER.debug("PLANNER = {}; TICK = {}/{}; PHASE = {}; COST = {}",
          this, cumulativeTicks, tick, phase.toString(), root.bestCost);

      VolcanoRuleMatch match = ruleQueue.popMatch(phase);
      if (match == null) {
        break;
      }

      assert match.getRule().matches(match);
      match.onMatch();

      // The root may have been merged with another
      // subset. Find the new root subset.
      root = canonize(root);
    }

    ruleQueue.phaseCompleted(phase);
  }
  if (LOGGER.isTraceEnabled()) {
    StringWriter sw = new StringWriter();
    final PrintWriter pw = new PrintWriter(sw);
    dump(pw);
    pw.flush();
    LOGGER.trace(sw.toString());
  }
  RelNode cheapest = root.buildCheapestPlan(this);
  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug(
        "Cheapest plan:\n{}", RelOptUtil.toString(cheapest, SqlExplainLevel.ALL_ATTRIBUTES));

    LOGGER.debug("Provenance:\n{}", provenance(cheapest));
  }
  return cheapest;
}
```


```
RelSet
/**
 * A <code>RelSet</code> is an equivalence-set of expressions; that is, a set of
 * expressions which have identical semantics. We are generally interested in
 * using the expression which has the lowest cost.
 *
 * <p>All of the expressions in an <code>RelSet</code> have the same calling
 * convention.</p>
 */
 
 /**
 * Subset of an equivalence class where all relational expressions have the
 * same physical properties.
 *
 * <p>Physical properties are instances of the {@link RelTraitSet}, and consist
 * of traits such as calling convention and collation (sort-order).
 *
 * <p>For some traits, a relational expression can have more than one instance.
 * For example, R can be sorted on both [X] and [Y, Z]. In which case, R would
 * belong to the sub-sets for [X] and [Y, Z]; and also the leading edges [Y] and
 * [].
 *
 * @see RelNode
 * @see RelSet
 * @see RelTrait
 */
public class RelSubset extends AbstractRelNode { // 也是一个RelNode
```



calcite 优化分这几个阶段：

```
PRE_PROCESS_MDR, PRE_PROCESS, OPTIMIZE, CLEANUP,
```

```
/**
 * Holds rule calls waiting to be fired.
 */
final RuleQueue ruleQueue = new RuleQueue(this);
```

```
VolcanoRuleMatch match = ruleQueue.popMatch(phase);
if (match == null) if (match == null) {
  break  break;
}
assert match.getRule().matches(match)assert match.getRule().matches(match);
match.onMatch()match.onMatch();
```

每个 RelSubset 在rulequeue里面有个重要性的概念，何解？