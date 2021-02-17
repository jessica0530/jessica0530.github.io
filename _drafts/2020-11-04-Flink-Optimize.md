---
layout: post
title: Flink Optimize
categories: [flink]
description: Flink Optimize
keywords: flink
---

# Flink 最新版本 

优化

````java

private List<Transformation<?>> translate(List<ModifyOperation> modifyOperations) {
		return planner.translate(modifyOperations);
	}

````

```java
 override def translate(
      tableOperations: util.List[ModifyOperation]): util.List[Transformation[_]] = {
    val planner = createDummyPlanner()
    tableOperations.asScala.map { operation =>
      val (ast, updatesAsRetraction) = translateToRel(operation)
      val optimizedPlan = optimizer.optimize(ast, updatesAsRetraction, getRelBuilder)
      val dataStream = translateToCRow(planner, optimizedPlan)
      dataStream.getTransformation.asInstanceOf[Transformation[_]]
    }.filter(Objects.nonNull).asJava
  }
```

```java
  /**
    * Generates the optimized [[RelNode]] tree from the original relational node tree.
    *
    * @param relNode The root node of the relational expression tree.
    * @param updatesAsRetraction True if the sink requests updates as retraction messages.
    * @return The optimized [[RelNode]] tree
    */
  def optimize(
    relNode: RelNode,
    updatesAsRetraction: Boolean,
    relBuilder: RelBuilder): RelNode = {
    
    
    //基于Rule的BOTTOM_UP 子查询的优化 SubQueryFilterRemoveRule,ProjectRemove,JoinRemove
    val convSubQueryPlan = optimizeConvertSubQueries(relNode)
   
    // 通过将对表的引用替换为适当的计划子树来扩展计划。那些规则可以创建新的计划节点 
   //基于TOP_DOWN 展开   
    val expandedPlan = optimizeExpandPlan(convSubQueryPlan)
      
   //  RelDecorrelator将关系表达式（RelNode）树中的所有相关表达式（corExp）替换为通过将生成corExp的RelNode与引用它的RelNode联接在一起而生成的非相关表达式。     
    val decorPlan = RelDecorrelator.decorrelateQuery(expandedPlan, relBuilder)
      
  //  转化time修饰符，通过RelTimeIndicatorConverter实现    
    val planWithMaterializedTimeAttributes =
      RelTimeIndicatorConverter.convert(decorPlan, relBuilder.getRexBuilder)
 // norm rule set   
 // 1.Transform window to LogicalWindowAggregate  
 // 2.simplify expressions rules
 // 3.谓词 合并到 in or not_in  过滤算子（就是你在sql语句里面写的where语句），尽可能地放在in or not_in里面   
    val normalizedPlan = optimizeNormalizeLogicalPlan(planWithMaterializedTimeAttributes)
 // logical_opt_rules
 // 1.38个 transformation rule 和 17个 converter 成 flinkLogical     
    val logicalPlan = optimizeLogicalPlan(normalizedPlan)
  //rewrite logical rules    
    val logicalRewritePlan = optimizeLogicalRewritePlan(logicalPlan)
    
   //physicalOptRuleSet - DataStreamConvertor CBO   
      val physicalPlan = optimizePhysicalPlan(logicalRewritePlan, FlinkConventions.DATASTREAM)
    
    // decorate  retract相关的  根据 updateAsRetraction
    // Trait赋值 updateAsRetraction 
      optimizeDecoratePlan(physicalPlan, updatesAsRetraction)
      
    
  }
```

# Blink增加的优化

```java
public class OptimizerConfigOptions {

	// ------------------------------------------------------------------------
	//  Optimizer Options
	// ------------------------------------------------------------------------
	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH_STREAMING)
	public static final ConfigOption<String> TABLE_OPTIMIZER_AGG_PHASE_STRATEGY =
		key("table.optimizer.agg-phase-strategy")
			.defaultValue("AUTO")
			.withDescription("Strategy for aggregate phase. Only AUTO, TWO_PHASE or ONE_PHASE can be set.\n" +
				"AUTO: No special enforcer for aggregate stage. Whether to choose two stage aggregate or one" +
				" stage aggregate depends on cost. \n" +
				"TWO_PHASE: Enforce to use two stage aggregate which has localAggregate and globalAggregate. " +
				"Note that if aggregate call does not support optimize into two phase, we will still use one stage aggregate.\n" +
				"ONE_PHASE: Enforce to use one stage aggregate which only has CompleteGlobalAggregate.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH)
	public static final ConfigOption<Long> TABLE_OPTIMIZER_BROADCAST_JOIN_THRESHOLD =
		key("table.optimizer.join.broadcast-threshold")
			.defaultValue(1024 * 1024L)
			.withDescription("Configures the maximum size in bytes for a table that will be broadcast to all worker " +
				"nodes when performing a join. By setting this value to -1 to disable broadcasting.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.STREAMING)
	public static final ConfigOption<Boolean> TABLE_OPTIMIZER_DISTINCT_AGG_SPLIT_ENABLED =
		key("table.optimizer.distinct-agg.split.enabled")
			.defaultValue(false)
			.withDescription("Tells the optimizer whether to split distinct aggregation " +
				"(e.g. COUNT(DISTINCT col), SUM(DISTINCT col)) into two level. " +
				"The first aggregation is shuffled by an additional key which is calculated using " +
				"the hashcode of distinct_key and number of buckets. This optimization is very useful " +
				"when there is data skew in distinct aggregation and gives the ability to scale-up the job. " +
				"Default is false.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.STREAMING)
	public static final ConfigOption<Integer> TABLE_OPTIMIZER_DISTINCT_AGG_SPLIT_BUCKET_NUM =
		key("table.optimizer.distinct-agg.split.bucket-num")
			.defaultValue(1024)
			.withDescription("Configure the number of buckets when splitting distinct aggregation. " +
				"The number is used in the first level aggregation to calculate a bucket key " +
				"'hash_code(distinct_key) % BUCKET_NUM' which is used as an additional group key after splitting.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH_STREAMING)
	public static final ConfigOption<Boolean> TABLE_OPTIMIZER_REUSE_SUB_PLAN_ENABLED =
		key("table.optimizer.reuse-sub-plan-enabled")
			.defaultValue(true)
			.withDescription("When it is true, the optimizer will try to find out duplicated sub-plans and reuse them.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH_STREAMING)
	public static final ConfigOption<Boolean> TABLE_OPTIMIZER_REUSE_SOURCE_ENABLED =
		key("table.optimizer.reuse-source-enabled")
			.defaultValue(true)
			.withDescription("When it is true, the optimizer will try to find out duplicated table sources and " +
				"reuse them. This works only when " + TABLE_OPTIMIZER_REUSE_SUB_PLAN_ENABLED.key() + " is true.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH_STREAMING)
	public static final ConfigOption<Boolean> TABLE_OPTIMIZER_SOURCE_PREDICATE_PUSHDOWN_ENABLED =
		key("table.optimizer.source.predicate-pushdown-enabled")
			.defaultValue(true)
			.withDescription("When it is true, the optimizer will push down predicates into the FilterableTableSource. " +
				"Default value is true.");

	@Documentation.TableOption(execMode = Documentation.ExecMode.BATCH_STREAMING)
	public static final ConfigOption<Boolean> TABLE_OPTIMIZER_JOIN_REORDER_ENABLED =
		key("table.optimizer.join-reorder-enabled")
			.defaultValue(false)
			.withDescription("Enables join reorder in optimizer. Default is disabled.");

```



# Match Type

## ARBITRARY

以任意顺序匹配。这是默认设置，因为它高效，并且大多数规则都不关心顺序。

## BOTTOM_UP

```
从叶子匹配。对后代的匹配尝试先于其祖先的所有匹配尝试
```

## TOP_DOWN

```
从根向下匹配。祖先的匹配尝试始终在其后代的所有匹配尝试之前
```

## DEPTH_FIRST

```
以深度优先顺序进行匹配。
它避免了在一个规则应用程序中生成新顶点后将规则重复应用于先前的RelNode。因此，在诸如带有大扇出的Union之类的情况下，它比ARBITRARY更有效
```