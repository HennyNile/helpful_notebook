# Postgres v15 Cost Model

## I. Cost Functions

### 1. Scan

```
cost_seqscan 
	Determines and returns the cost of scanning a relation sequentially.
cost_samplescan
	Determines and returns the cost of scanning a relation using sampling.
cost_bitmap_heap_scan
	Determines and returns the cost of scanning a relation using a bitmap index-then-heap plan.
cost_bitmap_tree_node
	Extract cost and selectivity from a bitmap tree node (index/and/or)
cost_bitmap_and_node
	Estimate the cost of a BitmapAnd node
cost_bitmap_or_node
	Estimate the cost of a BitmapOr node
cost_tidscan
	Determines and returns the cost of scanning a relation using TIDs.
cost_tidrangescan
	Determines and sets the costs of scanning a relation using a range of TIDs for 'path'
cost_namedtuplestorescan
	Determines and returns the cost of scanning a named tuplestore.
cost_index
	Determines and returns the cost of scanning a relation using an index.
cost_memoize_rescan
	Determines the estimated cost of rescanning a Memoize node.
cost_rescan
	Given a finished Path, estimate the costs of rescanning it after having done so the first time.  For some Path types a 
	rescan is cheaper than an original scan (if no parameters change), and this function embodies knowledge about that.  The 
	default is to return the same costs stored in the Path.  (Note that the cost estimates actually stored in Paths are 
	always for first scans.)

cost_subqueryscan
	Determines and returns the cost of scanning a subquery RTE.
cost_functionscan
	Determines and returns the cost of scanning a function RTE.
cost_tablefuncscan
	Determines and returns the cost of scanning a table function.
cost_valuesscan
	Determines and returns the cost of scanning a VALUES RTE.
cost_ctescan
	Determines and returns the cost of scanning a CTE RTE.
cost_resultscan
	Determines and returns the cost of scanning an RTE_RESULT relation.
```

From https://severalnines.com/blog/overview-various-scan-methods-postgresql/, Postgres supports below scan methods:

​	Sequential Scan

​	Index Scan

​	Index Only Scan

​	Bitmap Scan

​	TID Scan

### 2. Join

```
initial_cost_nestloop
	Preliminary estimate of the cost of a nestloop join path.
final_cost_nestloop
	Final estimate of the cost and result size of a nestloop join path.
initial_cost_mergejoin
	Preliminary estimate of the cost of a mergejoin path.
final_cost_mergejoin
	Final estimate of the cost and result size of a mergejoin path.
initial_cost_hashjoin
	Preliminary estimate of the cost of a hashjoin path.
final_cost_hashjoin
	Final estimate of the cost and result size of a hashjoin path.
```

### 3. Sort

```
cost_tuplesort
	Determines and returns the cost of sorting a relation using tuplesort, not including the cost of reading the input data.
cost_incremental_sort
	Determines and returns the cost of sorting a relation incrementally, when the input path is presorted by a prefix of the
	pathkeys.
cost_sort
	Determines and returns the cost of sorting a relation, including the cost of reading the input data.
```

### 4. Aggregate

```
cost_agg
	Determines and returns the cost of performing an Agg plan node, including the cost of its input.
cost_windowagg
	Determines and returns the cost of performing a WindowAgg plan node, including the cost of its input.
```

### 5. Append

```
cost_append
	Determines and returns the cost of an Append node.
cost_merge_append
	Determines and returns the cost of a MergeAppend node.
```

### 6. Group

```
cost_group
	Determines and returns the cost of performing a Group plan node, including the cost of its input.
```

### 7. Where

```
cost_qual_eval
	Estimate the CPU costs of evaluating a WHERE clause. The input can be either an implicitly-ANDed list of boolean 
	expressions, or a list of RestrictInfo nodes.  (The latter is preferred since it allows caching of the results.) The 
	result includes both a one-time (startup) component, and a per-evaluation component.
cost_qual_eval_node
	As above, for a single RestrictInfo or expression.
cost_qual_eval_walker
```

### Others

```
cost_gather
	Determines and returns the cost of gather path.
cost_gather_merge
	Determines and returns the cost of gather merge path.

cost_recursive_union
	Determines and returns the cost of performing a recursive union, and also the estimated output size.
append_nonpartial_cost
	Estimate the cost of the non-partial paths in a Parallel Append. The non-partial paths are assumed to be the first 
	"numpaths" paths	from the subpaths list, and to be in order of decreasing cost.
cost_material
	Determines and returns the cost of materializing a relation, including the cost of reading the input data.
cost_subplan
	Figure the costs for a SubPlan (or initplan).
```