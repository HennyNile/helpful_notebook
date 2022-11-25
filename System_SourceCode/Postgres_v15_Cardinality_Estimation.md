# Postgres v15 Cardinality Estimation

In this doc, we will demonstrate three things related to cardinality in postgres optimizer.

## N. Important Types

From the code, we could see the cardinality of relation are related to several types (**PlannerInfo**, **ParamPathInfo**, **ReloptInfo**), then we just list definitions of these types for easier view.

### 1. PlannerInfo

```c
src/include/nodes/pathnodes.h
struct PlannerInfo
{
	NodeTag		type;

	Query	   *parse;			/* the Query being planned */

	PlannerGlobal *glob;		/* global info for current planner run */

	Index		query_level;	/* 1 at the outermost Query */

	PlannerInfo *parent_root;	/* NULL at outermost Query */

	/*
	 * plan_params contains the expressions that this query level needs to
	 * make available to a lower query level that is currently being planned.
	 * outer_params contains the paramIds of PARAM_EXEC Params that outer
	 * query levels will make available to this query level.
	 */
	List	   *plan_params;	/* list of PlannerParamItems, see below */
	Bitmapset  *outer_params;

	/*
	 * simple_rel_array holds pointers to "base rels" and "other rels" (see
	 * comments for RelOptInfo for more info).  It is indexed by rangetable
	 * index (so entry 0 is always wasted).  Entries can be NULL when an RTE
	 * does not correspond to a base relation, such as a join RTE or an
	 * unreferenced view RTE; or if the RelOptInfo hasn't been made yet.
	 */
	struct RelOptInfo **simple_rel_array;	/* All 1-rel RelOptInfos */
	int			simple_rel_array_size;	/* allocated size of array */

	/*
	 * simple_rte_array is the same length as simple_rel_array and holds
	 * pointers to the associated rangetable entries.  Using this is a shade
	 * faster than using rt_fetch(), mostly due to fewer indirections.
	 */
	RangeTblEntry **simple_rte_array;	/* rangetable as an array */

	/*
	 * append_rel_array is the same length as the above arrays, and holds
	 * pointers to the corresponding AppendRelInfo entry indexed by
	 * child_relid, or NULL if the rel is not an appendrel child.  The array
	 * itself is not allocated if append_rel_list is empty.
	 */
	struct AppendRelInfo **append_rel_array;

	/*
	 * all_baserels is a Relids set of all base relids (but not "other"
	 * relids) in the query; that is, the Relids identifier of the final join
	 * we need to form.  This is computed in make_one_rel, just before we
	 * start making Paths.
	 */
	Relids		all_baserels;

	/*
	 * nullable_baserels is a Relids set of base relids that are nullable by
	 * some outer join in the jointree; these are rels that are potentially
	 * nullable below the WHERE clause, SELECT targetlist, etc.  This is
	 * computed in deconstruct_jointree.
	 */
	Relids		nullable_baserels;

	/*
	 * join_rel_list is a list of all join-relation RelOptInfos we have
	 * considered in this planning run.  For small problems we just scan the
	 * list to do lookups, but when there are many join relations we build a
	 * hash table for faster lookups.  The hash table is present and valid
	 * when join_rel_hash is not NULL.  Note that we still maintain the list
	 * even when using the hash table for lookups; this simplifies life for
	 * GEQO.
	 */
	List	   *join_rel_list;	/* list of join-relation RelOptInfos */
	struct HTAB *join_rel_hash; /* optional hashtable for join relations */

	/*
	 * When doing a dynamic-programming-style join search, join_rel_level[k]
	 * is a list of all join-relation RelOptInfos of level k, and
	 * join_cur_level is the current level.  New join-relation RelOptInfos are
	 * automatically added to the join_rel_level[join_cur_level] list.
	 * join_rel_level is NULL if not in use.
	 */
	List	  **join_rel_level; /* lists of join-relation RelOptInfos */
	int			join_cur_level; /* index of list being extended */

	List	   *init_plans;		/* init SubPlans for query */

	List	   *cte_plan_ids;	/* per-CTE-item list of subplan IDs (or -1 if
								 * no subplan was made for that CTE) */

	List	   *multiexpr_params;	/* List of Lists of Params for MULTIEXPR
									 * subquery outputs */

	List	   *eq_classes;		/* list of active EquivalenceClasses */

	bool		ec_merging_done;	/* set true once ECs are canonical */

	List	   *canon_pathkeys; /* list of "canonical" PathKeys */

	List	   *left_join_clauses;	/* list of RestrictInfos for mergejoinable
									 * outer join clauses w/nonnullable var on
									 * left */

	List	   *right_join_clauses; /* list of RestrictInfos for mergejoinable
									 * outer join clauses w/nonnullable var on
									 * right */

	List	   *full_join_clauses;	/* list of RestrictInfos for mergejoinable
									 * full join clauses */

	List	   *join_info_list; /* list of SpecialJoinInfos */

	/*
	 * all_result_relids is empty for SELECT, otherwise it contains at least
	 * parse->resultRelation.  For UPDATE/DELETE/MERGE across an inheritance
	 * or partitioning tree, the result rel's child relids are added.  When
	 * using multi-level partitioning, intermediate partitioned rels are
	 * included. leaf_result_relids is similar except that only actual result
	 * tables, not partitioned tables, are included in it.
	 */
	Relids		all_result_relids;	/* set of all result relids */
	Relids		leaf_result_relids; /* set of all leaf relids */

	/*
	 * Note: for AppendRelInfos describing partitions of a partitioned table,
	 * we guarantee that partitions that come earlier in the partitioned
	 * table's PartitionDesc will appear earlier in append_rel_list.
	 */
	List	   *append_rel_list;	/* list of AppendRelInfos */

	List	   *row_identity_vars;	/* list of RowIdentityVarInfos */

	List	   *rowMarks;		/* list of PlanRowMarks */

	List	   *placeholder_list;	/* list of PlaceHolderInfos */

	List	   *fkey_list;		/* list of ForeignKeyOptInfos */

	List	   *query_pathkeys; /* desired pathkeys for query_planner() */

	List	   *group_pathkeys; /* groupClause pathkeys, if any */
	List	   *window_pathkeys;	/* pathkeys of bottom window, if any */
	List	   *distinct_pathkeys;	/* distinctClause pathkeys, if any */
	List	   *sort_pathkeys;	/* sortClause pathkeys, if any */

	List	   *part_schemes;	/* Canonicalised partition schemes used in the
								 * query. */

	List	   *initial_rels;	/* RelOptInfos we are now trying to join */

	/* Use fetch_upper_rel() to get any particular upper rel */
	List	   *upper_rels[UPPERREL_FINAL + 1]; /* upper-rel RelOptInfos */

	/* Result tlists chosen by grouping_planner for upper-stage processing */
	struct PathTarget *upper_targets[UPPERREL_FINAL + 1];

	/*
	 * The fully-processed targetlist is kept here.  It differs from
	 * parse->targetList in that (for INSERT) it's been reordered to match the
	 * target table, and defaults have been filled in.  Also, additional
	 * resjunk targets may be present.  preprocess_targetlist() does most of
	 * that work, but note that more resjunk targets can get added during
	 * appendrel expansion.  (Hence, upper_targets mustn't get set up till
	 * after that.)
	 */
	List	   *processed_tlist;

	/*
	 * For UPDATE, this list contains the target table's attribute numbers to
	 * which the first N entries of processed_tlist are to be assigned.  (Any
	 * additional entries in processed_tlist must be resjunk.)  DO NOT use the
	 * resnos in processed_tlist to identify the UPDATE target columns.
	 */
	List	   *update_colnos;

	/* Fields filled during create_plan() for use in setrefs.c */
	AttrNumber *grouping_map;	/* for GroupingFunc fixup */
	List	   *minmax_aggs;	/* List of MinMaxAggInfos */

	MemoryContext planner_cxt;	/* context holding PlannerInfo */

	Cardinality total_table_pages;	/* # of pages in all non-dummy tables of
									 * query */

	Selectivity tuple_fraction; /* tuple_fraction passed to query_planner */
	Cardinality limit_tuples;	/* limit_tuples passed to query_planner */

	Index		qual_security_level;	/* minimum security_level for quals */
	/* Note: qual_security_level is zero if there are no securityQuals */

	bool		hasJoinRTEs;	/* true if any RTEs are RTE_JOIN kind */
	bool		hasLateralRTEs; /* true if any RTEs are marked LATERAL */
	bool		hasHavingQual;	/* true if havingQual was non-null */
	bool		hasPseudoConstantQuals; /* true if any RestrictInfo has
										 * pseudoconstant = true */
	bool		hasAlternativeSubPlans; /* true if we've made any of those */
	bool		hasRecursion;	/* true if planning a recursive WITH item */

	/*
	 * Information about aggregates. Filled by preprocess_aggrefs().
	 */
	List	   *agginfos;		/* AggInfo structs */
	List	   *aggtransinfos;	/* AggTransInfo structs */
	int			numOrderedAggs; /* number w/ DISTINCT/ORDER BY/WITHIN GROUP */
	bool		hasNonPartialAggs;	/* does any agg not support partial mode? */
	bool		hasNonSerialAggs;	/* is any partial agg non-serializable? */

	/* These fields are used only when hasRecursion is true: */
	int			wt_param_id;	/* PARAM_EXEC ID for the work table */
	struct Path *non_recursive_path;	/* a path for non-recursive term */

	/* These fields are workspace for createplan.c */
	Relids		curOuterRels;	/* outer rels above current node */
	List	   *curOuterParams; /* not-yet-assigned NestLoopParams */

	/* These fields are workspace for setrefs.c */
	bool	   *isAltSubplan;	/* array corresponding to glob->subplans */
	bool	   *isUsedSubplan;	/* array corresponding to glob->subplans */

	/* optional private data for join_search_hook, e.g., GEQO */
	void	   *join_search_private;

	/* Does this query modify any partition key columns? */
	bool		partColsUpdated;
};
```

### 2. ParamPathInfo

```c
src/include/nodes/pathnodes.h
/*
 * ParamPathInfo
 *
 * All parameterized paths for a given relation with given required outer rels
 * link to a single ParamPathInfo, which stores common information such as
 * the estimated rowcount for this parameterization.  We do this partly to
 * avoid recalculations, but mostly to ensure that the estimated rowcount
 * is in fact the same for every such path.
 *
 * Note: ppi_clauses is only used in ParamPathInfos for base relation paths;
 * in join cases it's NIL because the set of relevant clauses varies depending
 * on how the join is formed.  The relevant clauses will appear in each
 * parameterized join path's joinrestrictinfo list, instead.
 */
typedef struct ParamPathInfo
{
	NodeTag		type;

	Relids		ppi_req_outer;	/* rels supplying parameters used by path */
	Cardinality ppi_rows;		/* estimated number of result tuples */
	List	   *ppi_clauses;	/* join clauses available from outer rels */
} ParamPathInfo;
```

### 3. ReloptInfo 

```c
src/include/nodes/pathnodes.h
/* he optimizer builds a RelOptInfo structure for each base relation used in the query.  Base rels are either primitive tables, or subquery subselects that are planned via a separate recursive invocation of the planner.  A RelOptInfo is also built for each join relation that is considered during planning.  A join rel is simply a combination of base rels.  There is only one join RelOptInfo for any given set of baserels --- for example, the join {A B C} is represented by the same RelOptInfo no matter whether we build it by joining A and B first and then adding C, or joining B and C first and then adding A, etc.  These different means of building the joinrel are represented as Paths.  For each RelOptInfo we build a list of Paths that represent plausible ways to implement the scan or join of that relation. Once we've considered all the plausible Paths for a rel, we select the one that is cheapest according to the planner's cost estimates.  The final plan is derived from the cheapest Path for the RelOptInfo that includes all the base rels of the query.
*/
typedef struct RelOptInfo
{
	NodeTag		type;

	RelOptKind	reloptkind;

	/* all relations included in this RelOptInfo */
    // typedef Bitmapset *Relids
	Relids		relids;			/* set of base relids (rangetable indexes) */

	/* size estimates generated by planner */
	Cardinality rows;			/* estimated number of result tuples */

	/* per-relation planner control flags */
	bool		consider_startup;	/* keep cheap-startup-cost paths? */
	bool		consider_param_startup; /* ditto, for parameterized paths? */
	bool		consider_parallel;	/* consider parallel paths? */

	/* default result targetlist for Paths scanning this relation */
	struct PathTarget *reltarget;	/* list of Vars/Exprs, cost, width */

	/* materialization information */
	List	   *pathlist;		/* Path structures */
	List	   *ppilist;		/* ParamPathInfos used in pathlist */
	List	   *partial_pathlist;	/* partial Paths */
	struct Path *cheapest_startup_path;
	struct Path *cheapest_total_path;
	struct Path *cheapest_unique_path;
	List	   *cheapest_parameterized_paths;

	/* parameterization information needed for both base rels and join rels */
	/* (see also lateral_vars and lateral_referencers) */
	Relids		direct_lateral_relids;	/* rels directly laterally referenced */
	Relids		lateral_relids; /* minimum parameterization of rel */

	/* information about a base rel (not set for join rels!) */
	Index		relid;
	Oid			reltablespace;	/* containing tablespace */
	RTEKind		rtekind;		/* RELATION, SUBQUERY, FUNCTION, etc */
	AttrNumber	min_attr;		/* smallest attrno of rel (often <0) */
	AttrNumber	max_attr;		/* largest attrno of rel */
	Relids	   *attr_needed;	/* array indexed [min_attr .. max_attr] */
	int32	   *attr_widths;	/* array indexed [min_attr .. max_attr] */
	List	   *lateral_vars;	/* LATERAL Vars and PHVs referenced by rel */
	Relids		lateral_referencers;	/* rels that reference me laterally */
	List	   *indexlist;		/* list of IndexOptInfo */
	List	   *statlist;		/* list of StatisticExtInfo */
	BlockNumber pages;			/* size estimates derived from pg_class */
	Cardinality tuples;
	double		allvisfrac;
	Bitmapset  *eclass_indexes; /* Indexes in PlannerInfo's eq_classes list of
								 * ECs that mention this rel */
	PlannerInfo *subroot;		/* if subquery */
	List	   *subplan_params; /* if subquery */
	int			rel_parallel_workers;	/* wanted number of parallel workers */
	uint32		amflags;		/* Bitmask of optional features supported by
								 * the table AM */

	/* Information about foreign tables and foreign joins */
	Oid			serverid;		/* identifies server for the table or join */
	Oid			userid;			/* identifies user to check access as */
	bool		useridiscurrent;	/* join is only valid for current user */
	/* use "struct FdwRoutine" to avoid including fdwapi.h here */
	struct FdwRoutine *fdwroutine;
	void	   *fdw_private;

	/* cache space for remembering if we have proven this relation unique */
	List	   *unique_for_rels;	/* known unique for these other relid
									 * set(s) */
	List	   *non_unique_for_rels;	/* known not unique for these set(s) */

	/* used by various scans and joins: */
	List	   *baserestrictinfo;	/* RestrictInfo structures (if base rel) */
	QualCost	baserestrictcost;	/* cost of evaluating the above */
	Index		baserestrict_min_security;	/* min security_level found in
											 * baserestrictinfo */
	List	   *joininfo;		/* RestrictInfo structures for join clauses
								 * involving this rel */
	bool		has_eclass_joins;	/* T means joininfo is incomplete */

	/* used by partitionwise joins: */
	bool		consider_partitionwise_join;	/* consider partitionwise join
												 * paths? (if partitioned rel) */
	Relids		top_parent_relids;	/* Relids of topmost parents (if "other"
									 * rel) */

	/* used for partitioned relations: */
	PartitionScheme part_scheme;	/* Partitioning scheme */
	int			nparts;			/* Number of partitions; -1 if not yet set; in
								 * case of a join relation 0 means it's
								 * considered unpartitioned */
	struct PartitionBoundInfoData *boundinfo;	/* Partition bounds */
	bool		partbounds_merged;	/* True if partition bounds were created
									 * by partition_bounds_merge() */
	List	   *partition_qual; /* Partition constraint, if not the root */
	struct RelOptInfo **part_rels;	/* Array of RelOptInfos of partitions,
									 * stored in the same order as bounds */
	Bitmapset  *live_parts;		/* Bitmap with members acting as indexes into
								 * the part_rels[] array to indicate which
								 * partitions survived partition pruning. */
	Relids		all_partrels;	/* Relids set of all partition relids */
	List	  **partexprs;		/* Non-nullable partition key expressions */
	List	  **nullable_partexprs; /* Nullable partition key expressions */
} RelOptInfo;
```

### 4. Relids (Bitmapset)

```c
// typedef Bitmapset *Relids
typedef struct Bitmapset
{
	int			nwords;			/* number of words in array */
	bitmapword	words[FLEXIBLE_ARRAY_MEMBER];	/* really [nwords] */
} Bitmapset;
```

## I. Cardinality of Base Relations

### 1. Computing Flow of Cardinality Estimation

```c
src/backend/optimizer/plan/planmain.c
/*
 * query_planner
 *	  Generate a path (that is, a simplified plan) for a basic query,
 *	  which may involve joins but not any fancier features.
 *
 * Since query_planner does not handle the toplevel processing (grouping,
 * sorting, etc) it cannot select the best path by itself.  Instead, it
 * returns the RelOptInfo for the top level of joining, and the caller
 * (grouping_planner) can choose among the surviving paths for the rel.
 *
 * root describes the query to plan
 * qp_callback is a function to compute query_pathkeys once it's safe to do so
 * qp_extra is optional extra data to pass to qp_callback
 *
 * Note: the PlannerInfo node also includes a query_pathkeys field, which
 * tells query_planner the sort order that is desired in the final output
 * plan.  This value is *not* available at call time, but is computed by
 * qp_callback once we have completed merging the query's equivalence classes.
 * (We cannot construct canonical pathkeys until that's done.)
 */
RelOptInfo *
query_planner(PlannerInfo *root,
			  query_pathkeys_callback qp_callback, void *qp_extra)
    /*
	 * Ready to do the primary planning.
	 */
	final_rel = make_one_rel(root, joinlist);

src/backend/optimizer/path/allpaths.c
/*
 * make_one_rel
 *	  Finds all possible access paths for executing a query, returning a
 *	  single rel that represents the join of all base rels in the query.
 */
RelOptInfo *
make_one_rel(PlannerInfo *root, List *joinlist)
    /*
	 * Compute size estimates and consider_parallel flags for each base rel.
	 */
	set_base_rel_sizes(root);

src/backend/optimizer/path/allpaths.c
/*
 * set_base_rel_sizes
 *	  Set the size estimates (rows and widths) for each base-relation entry.
 *	  Also determine whether to consider parallel paths for base relations.
 *
 * We do this in a separate pass over the base rels so that rowcount
 * estimates are available for parameterized path generation, and also so
 * that each rel's consider_parallel flag is set correctly before we begin to
 * generate paths.
 */
static void
set_base_rel_sizes(PlannerInfo *root)
    for (rti = 1; rti < root->simple_rel_array_size; rti++)
	{
        RelOptInfo *rel = root->simple_rel_array[rti];
        RangeTblEntry *rte = root->simple_rte_array[rti];
    	set_rel_size(root, rel, rti, rte);
    }

src/backend/optimizer/path/allpaths.c
/*
 * set_rel_size
 *	  Set size estimates for a base relation
 */
static void
set_rel_size(PlannerInfo *root, RelOptInfo *rel,
			 Index rti, RangeTblEntry *rte)
    /* This method considers sizes of differet kinds of tables, such as 
   	 * foreign table, sampled table or plain table, we just foucs on 
     * plain table now.
     */
	set_plain_rel_size(root, rel, rte);

src/backend/optimizer/path/allpaths.c
/*
 * set_plain_rel_size
 *	  Set size estimates for a plain relation (no subquery, no inheritance)
 */
static void
set_plain_rel_size(PlannerInfo *root, RelOptInfo *rel, RangeTblEntry *rte)
	/* Mark rel with estimated output rows, width, etc */
	set_baserel_size_estimates(root, rel);

src/backend/optimizer/path/costsize.c
/*
 * set_baserel_size_estimates
 *		Set the size estimates for the given base relation.
 *
 * The rel's targetlist and restrictinfo list must have been constructed
 * already, and rel->tuples must be set.
 *
 * We set the following fields of the rel node:
 *	rows: the estimated number of output tuples (after applying
 *		  restriction clauses).
 *	width: the estimated average output tuple width in bytes.
 *	baserestrictcost: estimated cost of evaluating baserestrictinfo clauses.
 */
void
set_baserel_size_estimates(PlannerInfo *root, RelOptInfo *rel)
	nrows = rel->tuples *
		clauselist_selectivity(root,
							   rel->baserestrictinfo,
							   0,
							   JOIN_INNER,
							   NULL);
	rel->rows = clamp_row_est(nrows);

src/backend/optimizer/path/clausesel.c
/*
 * clauselist_selectivity -
 *	  Compute the selectivity of an implicitly-ANDed list of boolean
 *	  expression clauses.  The list can be empty, in which case 1.0
 *	  must be returned.  List elements may be either RestrictInfos
 *	  or bare expression clauses --- the former is preferred since
 *	  it allows caching of results.
 *
 * See clause_selectivity() for the meaning of the additional parameters.
 *
 * The basic approach is to apply extended statistics first, on as many
 * clauses as possible, in order to capture cross-column dependencies etc.
 * The remaining clauses are then estimated by taking the product of their
 * selectivities, but that's only right if they have independent
 * probabilities, and in reality they are often NOT independent even if they
 * only refer to a single column.  So, we want to be smarter where we can.
 *
 * We also recognize "range queries", such as "x > 34 AND x < 42".  Clauses
 * are recognized as possible range query components if they are restriction
 * opclauses whose operators have scalarltsel or a related function as their
 * restriction selectivity estimator.  We pair up clauses of this form that
 * refer to the same variable.  An unpairable clause of this kind is simply
 * multiplied into the selectivity product in the normal way.  But when we
 * find a pair, we know that the selectivities represent the relative
 * positions of the low and high bounds within the column's range, so instead
 * of figuring the selectivity as hisel * losel, we can figure it as hisel +
 * losel - 1.  (To visualize this, see that hisel is the fraction of the range
 * below the high bound, while losel is the fraction above the low bound; so
 * hisel can be interpreted directly as a 0..1 value but we need to convert
 * losel to 1-losel before interpreting it as a value.  Then the available
 * range is 1-losel to hisel.  However, this calculation double-excludes
 * nulls, so really we need hisel + losel + null_frac - 1.)
 *
 * If either selectivity is exactly DEFAULT_INEQ_SEL, we forget this equation
 * and instead use DEFAULT_RANGE_INEQ_SEL.  The same applies if the equation
 * yields an impossible (negative) result.
 *
 * A free side-effect is that we can recognize redundant inequalities such
 * as "x < 4 AND x < 5"; only the tighter constraint will be counted.
 *
 * Of course this is all very dependent on the behavior of the inequality
 * selectivity functions; perhaps some day we can generalize the approach.
 */
Selectivity
clauselist_selectivity(PlannerInfo *root,
					   List *clauses,
					   int varRelid,
					   JoinType jointype,
					   SpecialJoinInfo *sjinfo)
{
	return clauselist_selectivity_ext(root, clauses, varRelid, jointype, sjinfo, true);
}

src/backend/optimizer/path/clausesel.c
/*
 * clauselist_selectivity_ext -
 *	  Extended version of clauselist_selectivity().  If "use_extended_stats"
 *	  is false, all extended statistics will be ignored, and only per-column
 *	  statistics will be used.
 */
Selectivity
clauselist_selectivity_ext(PlannerInfo *root,
						   List *clauses,
						   int varRelid,
						   JoinType jointype,
						   SpecialJoinInfo *sjinfo,
						   bool use_extended_stats)
	if (list_length(clauses) == 1)
		return clause_selectivity_ext(root, (Node *) linitial(clauses),
									  varRelid, jointype, sjinfo,
									  use_extended_stats);

src/backend/optimizer/path/clausesel.c
/*
 * clause_selectivity_ext -
 *	  Extended version of clause_selectivity().  If "use_extended_stats" is
 *	  false, all extended statistics will be ignored, and only per-column
 *	  statistics will be used.
 */
Selectivity
clause_selectivity_ext(PlannerInfo *root,
					   Node *clause,
					   int varRelid,
					   JoinType jointype,
					   SpecialJoinInfo *sjinfo,
					   bool use_extended_stats)
    /* Estimate selectivity for a restriction clause. */
    s1 = restriction_selectivity(root, opno,
                                 opclause->args,
                                 opclause->inputcollid,
                                 varRelid);

src/backend/optimizer/util/plancat.c
/*
 * restriction_selectivity
 *
 * Returns the selectivity of a specified restriction operator clause.
 * This code executes registered procedures stored in the
 * operator relation, by calling the function manager.
 *
 * See clause_selectivity() for the meaning of the additional parameters.
 */
Selectivity
restriction_selectivity(PlannerInfo *root,
						Oid operatorid,
						List *args,
						Oid inputcollid,
						int varRelid)
    result = DatumGetFloat8(OidFunctionCall4Coll(oprrest,
												 inputcollid,
												 PointerGetDatum(root),
												 ObjectIdGetDatum(operatorid),
												 PointerGetDatum(args),
												 Int32GetDatum(varRelid)));

src/backend/utils/fmgr/fmgr.c
Datum
OidFunctionCall4Coll(Oid functionId, Oid collation, Datum arg1, Datum arg2,
					 Datum arg3, Datum arg4)
{
	FmgrInfo	flinfo;

	fmgr_info(functionId, &flinfo);

	return FunctionCall4Coll(&flinfo, collation, arg1, arg2, arg3, arg4);
}
```

### 2. The Representation of Predicates in Postgres

This part is to show how postgres stores predicates in code.

Use IMDb dataset and take the following query as the example.

```sql
SELECT count(*) 
FROM movie_info_idx mi_idx,
    title t
WHERE t.id=mi_idx.movie_id AND
    t.production_year=1977 AND 
    mi_idx.id<627517;
```

Here are three predicates ```t.id=mi_idx.movie_id```, ```t.production_year=1977```, ```mi_idx.id<627517```.

To estimate cardinality, Postgres first computes selectivity based on the predicates and then get result cardinality by multiplying the selectivity and cardinality of input relation. Postgres computes selectivity in the following method ```clause_selectivity_ext()```. 

```c
src/backend/optimizer/path/clausesel.c
/*
 * clause_selectivity_ext -
 *	  Extended version of clause_selectivity().  If "use_extended_stats" is
 *	  false, all extended statistics will be ignored, and only per-column
 *	  statistics will be used.
 */
Selectivity
clause_selectivity_ext(PlannerInfo *root,
					   Node *clause,
					   int varRelid,
					   JoinType jointype,
					   SpecialJoinInfo *sjinfo,
					   bool use_extended_stats)
    if (is_opclause(clause) || IsA(clause, DistinctExpr))
	{
		OpExpr	   *opclause = (OpExpr *) clause;
		Oid			opno = opclause->opno;

		if (treat_as_join_clause(root, clause, rinfo, varRelid, sjinfo))
		{
			/* Estimate selectivity for a join clause. */
			s1 = join_selectivity(root, opno,
								  opclause->args,
								  opclause->inputcollid,
								  jointype,
								  sjinfo);
		}
		else
		{
			/* Estimate selectivity for a restriction clause. */
			s1 = restriction_selectivity(root, opno,
										 opclause->args,
										 opclause->inputcollid,
										 varRelid);
		}

		/*
		 * DistinctExpr has the same representation as OpExpr, but the
		 * contained operator is "=" not "<>", so we must negate the result.
		 * This estimation method doesn't give the right behavior for nulls,
		 * but it's better than doing nothing.
		 */
		if (IsA(clause, DistinctExpr))
			s1 = 1.0 - s1;
	}
```

 This method only computes the selectivity for one predicate which is transfered as parameter **Node *clause**.  We focus on predicate ```t.production_year=1977``` which has two arguments (t.production, 1977) and one operator (=). Two arguments are stored in ```opclause->args``` and the operator is stored in ```opclause->opno```. 

For arguments, columns are stored are variables with type **Var** and constants are stored as variables with type **Const**. You could get the certain variable via the following method

```c
foreach(arg, opclause->args)
{
    if (IsA(lfirst(arg), Const)) // get constant
    {
        Const* const_var = (Const *)lfirst(arg);
        long const_value = const_var->constvalue;
    }
    else if (IsA(lfirst(arg), Var)) // get column
    {
        Var* var = (Var *)lfirst(arg);
        int varattno = var->varattno; // varattno is the index of column in its relation
    }
}
```

For operators, you could get all operators' code in file **pg_operator.dat**. The operator id for = in ```t.production_year=1977``` is 96.

## II. Paths Generation of Base Relations

There are 12 kinds scan in postgres. They are **sequence scan**,  **sample scan**,  **bitmap heap scan**,  **tid scan**,  **tid range scan**,  **function scan**,  **table function scan**,  **value scan**,  **cte scan**,  **named tuplestore scan**,  **result scan**.

We take sequence scan as the example here.

```c
src/backend/optimizer/plan/planmain.c
/*
 * query_planner
 *	  Generate a path (that is, a simplified plan) for a basic query,
 *	  which may involve joins but not any fancier features.
 *
 * Since query_planner does not handle the toplevel processing (grouping,
 * sorting, etc) it cannot select the best path by itself.  Instead, it
 * returns the RelOptInfo for the top level of joining, and the caller
 * (grouping_planner) can choose among the surviving paths for the rel.
 *
 * root describes the query to plan
 * qp_callback is a function to compute query_pathkeys once it's safe to do so
 * qp_extra is optional extra data to pass to qp_callback
 *
 * Note: the PlannerInfo node also includes a query_pathkeys field, which
 * tells query_planner the sort order that is desired in the final output
 * plan.  This value is *not* available at call time, but is computed by
 * qp_callback once we have completed merging the query's equivalence classes.
 * (We cannot construct canonical pathkeys until that's done.)
 */
RelOptInfo *
query_planner(PlannerInfo *root,
			  query_pathkeys_callback qp_callback, void *qp_extra)
    /*
	 * Ready to do the primary planning.
	 */
	final_rel = make_one_rel(root, joinlist);

src/backend/optimizer/path/allpaths.c
/*
 * make_one_rel
 *	  Finds all possible access paths for executing a query, returning a
 *	  single rel that represents the join of all base rels in the query.
 */
RelOptInfo *
make_one_rel(PlannerInfo *root, List *joinlist)
    /*
	 * Generate access paths for each base rel.
	 */
	set_base_rel_pathlists(root);
	

src/backend/optimizer/path/allpaths.c
/*
 * set_base_rel_pathlists
 *	  Finds all paths available for scanning each base-relation entry.
 *	  Sequential scan and any available indices are considered.
 *	  Each useful path is attached to its relation's 'pathlist' field.
 */
static void
set_base_rel_pathlists(PlannerInfo *root)
{
	Index		rti;

	for (rti = 1; rti < root->simple_rel_array_size; rti++)
	{
		RelOptInfo *rel = root->simple_rel_array[rti];

		/* there may be empty slots corresponding to non-baserel RTEs */
		if (rel == NULL)
			continue;

		Assert(rel->relid == rti);	/* sanity check on array */

		/* ignore RTEs that are "other rels" */
		if (rel->reloptkind != RELOPT_BASEREL)
			continue;

		set_rel_pathlist(root, rel, rti, root->simple_rte_array[rti]);
	}
}

src/backend/optimizer/path/allpaths.c
static void
set_rel_pathlist(PlannerInfo *root, RelOptInfo *rel,
				 Index rti, RangeTblEntry *rte)
    set_plain_rel_pathlist(root, rel, rte);


src/backend/optimizer/path/allpaths.c
static void
set_plain_rel_pathlist(PlannerInfo *root, RelOptInfo *rel, RangeTblEntry *rte)
{
    Relids		required_outer;

	/*
	 * We don't support pushing join clauses into the quals of a seqscan, but
	 * it could still have required parameterization due to LATERAL refs in
	 * its tlist.
	 */
	required_outer = rel->lateral_relids;

	/* Consider sequential scan */
	add_path(rel, create_seqscan_path(root, rel, required_outer, 0));

	/* If appropriate, consider parallel sequential scan */
	if (rel->consider_parallel && required_outer == NULL)
		create_plain_partial_paths(root, rel);

	/* Consider index scans */
	create_index_paths(root, rel);

	/* Consider TID scans */
	create_tidscan_paths(root, rel);
}
    
src/backend/optimizer/util/pathnode.c
Path *
create_seqscan_path(PlannerInfo *root, RelOptInfo *rel, Relids required_outer, int parallel_workers)
    cost_seqscan(pathnode, root, rel, pathnode->param_info);

src/backend/optimizer/path/costsize.c
void cost_seqscan(Path *path, PlannerInfo *root, RelOptInfo *baserel, ParamPathInfo *param_info)
	/* Mark the path with the correct row estimate */
	if (param_info)
		path->rows = param_info->ppi_rows;
	else
		path->rows = baserel->rows;
```

## III. Cardinality of Join Operators

### 1. Computing Flow of Cardinality Estimation

```c
src/backend/optimizer/plan/planmain.c
/*
 * query_planner
 *	  Generate a path (that is, a simplified plan) for a basic query,
 *	  which may involve joins but not any fancier features.
 *
 * Since query_planner does not handle the toplevel processing (grouping,
 * sorting, etc) it cannot select the best path by itself.  Instead, it
 * returns the RelOptInfo for the top level of joining, and the caller
 * (grouping_planner) can choose among the surviving paths for the rel.
 *
 * root describes the query to plan
 * qp_callback is a function to compute query_pathkeys once it's safe to do so
 * qp_extra is optional extra data to pass to qp_callback
 *
 * Note: the PlannerInfo node also includes a query_pathkeys field, which
 * tells query_planner the sort order that is desired in the final output
 * plan.  This value is *not* available at call time, but is computed by
 * qp_callback once we have completed merging the query's equivalence classes.
 * (We cannot construct canonical pathkeys until that's done.)
 */
RelOptInfo *
query_planner(PlannerInfo *root,
			  query_pathkeys_callback qp_callback, void *qp_extra)
    /*
	 * Ready to do the primary planning.
	 */
	final_rel = make_one_rel(root, joinlist);

/*
 * make_one_rel
 *	  Finds all possible access paths for executing a query, returning a
 *	  single rel that represents the join of all base rels in the query.
 */
RelOptInfo *
make_one_rel(PlannerInfo *root, List *joinlist)
	rel = make_rel_from_joinlist(root, joinlist);

src/backend/optimizer/path/allpaths.c
/*
 * make_rel_from_joinlist
 *	  Build access paths using a "joinlist" to guide the join path search.
 *
 * See comments for deconstruct_jointree() for definition of the joinlist
 * data structure.
 */
static RelOptInfo *
make_rel_from_joinlist(PlannerInfo *root, List *joinlist)
    if (IsA(jlnode, List))
    {
        /* Recurse to handle subproblem */
        thisrel = make_rel_from_joinlist(root, (List *) jlnode);
    }
    
	return standard_join_search(root, levels_needed, initial_rels);

src/backend/optimizer/path/allpaths.c
/*
 * standard_join_search
 *	  Find possible joinpaths for a query by successively finding ways
 *	  to join component relations into join relations.
 *
 * 'levels_needed' is the number of iterations needed, ie, the number of
 *		independent jointree items in the query.  This is > 1.
 *
 * 'initial_rels' is a list of RelOptInfo nodes for each independent
 *		jointree item.  These are the components to be joined together.
 *		Note that levels_needed == list_length(initial_rels).
 *
 * Returns the final level of join relations, i.e., the relation that is
 * the result of joining all the original relations together.
 * At least one implementation path must be provided for this relation and
 * all required sub-relations.
 *
 * To support loadable plugins that modify planner behavior by changing the
 * join searching algorithm, we provide a hook variable that lets a plugin
 * replace or supplement this function.  Any such hook must return the same
 * final join relation as the standard code would, but it might have a
 * different set of implementation paths attached, and only the sub-joinrels
 * needed for these paths need have been instantiated.
 *
 * Note to plugin authors: the functions invoked during standard_join_search()
 * modify root->join_rel_list and root->join_rel_hash.  If you want to do more
 * than one join-order search, you'll probably need to save and restore the
 * original states of those data structures.  See geqo_eval() for an example.
 */
RelOptInfo *
standard_join_search(PlannerInfo *root, int levels_needed, List *initial_rels)
	for (lev = 2; lev <= levels_needed; lev++)
	{
		join_search_one_level(root, lev);
    }

src/backend/optimizer/path/joinrels.c
/*
 * join_search_one_level
 *	  Consider ways to produce join relations containing exactly 'level'
 *	  jointree items.  (This is one step of the dynamic-programming method
 *	  embodied in standard_join_search.)  Join rel nodes for each feasible
 *	  combination of lower-level rels are created and returned in a list.
 *	  Implementation paths are created for each such joinrel, too.
 *
 * level: level of rels we want to make this time
 * root->join_rel_level[j], 1 <= j < level, is a list of rels containing j items
 *
 * The result is returned in root->join_rel_level[level].
 */
void
join_search_one_level(PlannerInfo *root, int level)
    make_rels_by_clause_joins(root, old_rel, other_rels_list, other_rels);
	make_rels_by_clauseless_joins(root, old_rel,joinrels[1]);

	// both above methods also call method (void) make_join_rel() to make join relations
	(void) make_join_rel(root, old_rel, new_rel);

src/backend/optimizer/path/joinrels.c
RelOptInfo *
make_join_rel(PlannerInfo *root, RelOptInfo *rel1, RelOptInfo *rel2)
    joinrel = build_join_rel(root, joinrelids, rel1, rel2, sjinfo, &restrictlist);

src/backend/optimizer/util/relnode.c
RelOptInfo *
build_join_rel(PlannerInfo *root, Relids joinrelids, RelOptInfo *outer_rel, RelOptInfo *inner_rel, SpecialJoinInfo *sjinfo, List **restrictlist_ptr)
    /*
	 * Set estimates of the joinrel's size.
	 */
	set_joinrel_size_estimates(root, joinrel, outer_rel, inner_rel, sjinfo, restrictlist);

src/backend/optimizer/path/costsize.c
void
set_joinrel_size_estimates(PlannerInfo *root, RelOptInfo *rel,
						   RelOptInfo *outer_rel,
						   RelOptInfo *inner_rel,
						   SpecialJoinInfo *sjinfo,
						   List *restrictlist)
{
	rel->rows = calc_joinrel_size_estimate(root, el, uter_rel, nner_rel, uter_rel->rows, nner_rel->rows, jinfo, estrictlist);
}

src/backend/optimizer/path/costsize.c
static double
calc_joinrel_size_estimate(PlannerInfo *root,
						   RelOptInfo *joinrel,
						   RelOptInfo *outer_rel,
						   RelOptInfo *inner_rel,
						   double outer_rows,
						   double inner_rows,
						   SpecialJoinInfo *sjinfo,
						   List *restrictlist_in)
{	
    /* This apparently-useless variable dodges a compiler bug in VS2013: */
	List	   *restrictlist = restrictlist_in;
	JoinType	jointype = sjinfo->jointype;
	Selectivity fkselec;
	Selectivity jselec;
	Selectivity pselec;
	double		nrows;

	/*
	 * Compute joinclause selectivity.  Note that we are only considering
	 * clauses that become restriction clauses at this join level; we are not
	 * double-counting them because they were not considered in estimating the
	 * sizes of the component rels.
	 *
	 * First, see whether any of the joinclauses can be matched to known FK
	 * constraints.  If so, drop those clauses from the restrictlist, and
	 * instead estimate their selectivity using FK semantics.  (We do this
	 * without regard to whether said clauses are local or "pushed down".
	 * Probably, an FK-matching clause could never be seen as pushed down at
	 * an outer join, since it would be strict and hence would be grounds for
	 * join strength reduction.)  fkselec gets the net selectivity for
	 * FK-matching clauses, or 1.0 if there are none.
	 */
	fkselec = get_foreign_key_join_selectivity(root,
											   outer_rel->relids,
											   inner_rel->relids,
											   sjinfo,
											   &restrictlist);

	/*
	 * For an outer join, we have to distinguish the selectivity of the join's
	 * own clauses (JOIN/ON conditions) from any clauses that were "pushed
	 * down".  For inner joins we just count them all as joinclauses.
	 */
	if (IS_OUTER_JOIN(jointype))
	{
		List	   *joinquals = NIL;
		List	   *pushedquals = NIL;
		ListCell   *l;

		/* Grovel through the clauses to separate into two lists */
		foreach(l, restrictlist)
		{
			RestrictInfo *rinfo = lfirst_node(RestrictInfo, l);

			if (RINFO_IS_PUSHED_DOWN(rinfo, joinrel->relids))
				pushedquals = lappend(pushedquals, rinfo);
			else
				joinquals = lappend(joinquals, rinfo);
		}

		/* Get the separate selectivities */
		jselec = clauselist_selectivity(root,
										joinquals,
										0,
										jointype,
										sjinfo);
		pselec = clauselist_selectivity(root,
										pushedquals,
										0,
										jointype,
										sjinfo);

		/* Avoid leaking a lot of ListCells */
		list_free(joinquals);
		list_free(pushedquals);
	}
	else
	{
		jselec = clauselist_selectivity(root,
										restrictlist,
										0,
										jointype,
										sjinfo);
		pselec = 0.0;			/* not used, keep compiler quiet */
	}

	/*
	 * Basically, we multiply size of Cartesian product by selectivity.
	 *
	 * If we are doing an outer join, take that into account: the joinqual
	 * selectivity has to be clamped using the knowledge that the output must
	 * be at least as large as the non-nullable input.  However, any
	 * pushed-down quals are applied after the outer join, so their
	 * selectivity applies fully.
	 *
	 * For JOIN_SEMI and JOIN_ANTI, the selectivity is defined as the fraction
	 * of LHS rows that have matches, and we apply that straightforwardly.
	 */
	switch (jointype)
	{
		case JOIN_INNER:
			nrows = outer_rows * inner_rows * fkselec * jselec;
			/* pselec not used */
			break;
		case JOIN_LEFT:
			nrows = outer_rows * inner_rows * fkselec * jselec;
			if (nrows < outer_rows)
				nrows = outer_rows;
			nrows *= pselec;
			break;
		case JOIN_FULL:
			nrows = outer_rows * inner_rows * fkselec * jselec;
			if (nrows < outer_rows)
				nrows = outer_rows;
			if (nrows < inner_rows)
				nrows = inner_rows;
			nrows *= pselec;
			break;
		case JOIN_SEMI:
			nrows = outer_rows * fkselec * jselec;
			/* pselec not used */
			break;
		case JOIN_ANTI:
			nrows = outer_rows * (1.0 - fkselec * jselec);
			nrows *= pselec;
			break;
		default:
			/* other values not expected here */
			elog(ERROR, "unrecognized join type: %d", (int) jointype);
			nrows = 0;			/* keep compiler quiet */
			break;
	}

	return clamp_row_est(nrows);
}

src/backend/optimizer/path/clausesel.c
/*
 * clauselist_selectivity -
 *	  Compute the selectivity of an implicitly-ANDed list of boolean
 *	  expression clauses.  The list can be empty, in which case 1.0
 *	  must be returned.  List elements may be either RestrictInfos
 *	  or bare expression clauses --- the former is preferred since
 *	  it allows caching of results.
 *
 * See clause_selectivity() for the meaning of the additional parameters.
 *
 * The basic approach is to apply extended statistics first, on as many
 * clauses as possible, in order to capture cross-column dependencies etc.
 * The remaining clauses are then estimated by taking the product of their
 * selectivities, but that's only right if they have independent
 * probabilities, and in reality they are often NOT independent even if they
 * only refer to a single column.  So, we want to be smarter where we can.
 *
 * We also recognize "range queries", such as "x > 34 AND x < 42".  Clauses
 * are recognized as possible range query components if they are restriction
 * opclauses whose operators have scalarltsel or a related function as their
 * restriction selectivity estimator.  We pair up clauses of this form that
 * refer to the same variable.  An unpairable clause of this kind is simply
 * multiplied into the selectivity product in the normal way.  But when we
 * find a pair, we know that the selectivities represent the relative
 * positions of the low and high bounds within the column's range, so instead
 * of figuring the selectivity as hisel * losel, we can figure it as hisel +
 * losel - 1.  (To visualize this, see that hisel is the fraction of the range
 * below the high bound, while losel is the fraction above the low bound; so
 * hisel can be interpreted directly as a 0..1 value but we need to convert
 * losel to 1-losel before interpreting it as a value.  Then the available
 * range is 1-losel to hisel.  However, this calculation double-excludes
 * nulls, so really we need hisel + losel + null_frac - 1.)
 *
 * If either selectivity is exactly DEFAULT_INEQ_SEL, we forget this equation
 * and instead use DEFAULT_RANGE_INEQ_SEL.  The same applies if the equation
 * yields an impossible (negative) result.
 *
 * A free side-effect is that we can recognize redundant inequalities such
 * as "x < 4 AND x < 5"; only the tighter constraint will be counted.
 *
 * Of course this is all very dependent on the behavior of the inequality
 * selectivity functions; perhaps some day we can generalize the approach.
 */
Selectivity
clauselist_selectivity(PlannerInfo *root,
					   List *clauses,
					   int varRelid,
					   JoinType jointype,
					   SpecialJoinInfo *sjinfo)
{
	return clauselist_selectivity_ext(root, clauses, varRelid, jointype, sjinfo, true);
}

src/backend/optimizer/path/clausesel.c
/*
 * clauselist_selectivity_ext -
 *	  Extended version of clauselist_selectivity().  If "use_extended_stats"
 *	  is false, all extended statistics will be ignored, and only per-column
 *	  statistics will be used.
 */
Selectivity
clauselist_selectivity_ext(PlannerInfo *root,
						   List *clauses,
						   int varRelid,
						   JoinType jointype,
						   SpecialJoinInfo *sjinfo,
						   bool use_extended_stats)
	if (list_length(clauses) == 1)
		return clause_selectivity_ext(root, (Node *) linitial(clauses),
									  varRelid, jointype, sjinfo,
									  use_extended_stats);

src/backend/optimizer/path/clausesel.c
/*
 * clause_selectivity_ext -
 *	  Extended version of clause_selectivity().  If "use_extended_stats" is
 *	  false, all extended statistics will be ignored, and only per-column
 *	  statistics will be used.
 */
Selectivity
clause_selectivity_ext(PlannerInfo *root,
					   Node *clause,
					   int varRelid,
					   JoinType jointype,
					   SpecialJoinInfo *sjinfo,
					   bool use_extended_stats)
    /* Estimate selectivity for a restriction clause. */
    s1 = restriction_selectivity(root, opno,
                                 opclause->args,
                                 opclause->inputcollid,
                                 varRelid);

```

### 2. The Representation of Join relation

## IV. Paths Generation of Join Operators

There are 3 kinds of join operators **nestloop join**, **merge join** and **hash join**. We take hash join are the example here.

