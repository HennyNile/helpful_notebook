# Framework of Postgres v15 Optimizer

**Postgres' source code has extremely clear paths compared with Hive and Spark. **

Code of Postgres' optimizer is located in **src/backend/optimizer** where there are 5 directories (prep, path, plan, geqo and util) and **the top-level entry point of the planner/optimizer is over in planner.c**.

The following is [the official introduction of optimizer directory](https://github.com/postgres/postgres/tree/REL_15_STABLE/src/backend/optimizer).

```
These directories take the Query structure returned by the parser, and generate a plan used by the executor.  
/plan directory generates the actual output plan.
/path code generates all possible ways to join the tables.
/prep handles various preprocessing steps for special cases. 
/util is utility stuff.  
/geqo is the separate "genetic optimization" planner --- it does a semi-random search through the join tree space, rather than exhaustively considering all possible join trees.  (But each join considered by /geqo is given to /path to create paths for, so we consider all possible implementation paths for each specific join pair even in GEQO mode.)
```

Then let us dive into each subdirectories.

## /plan

This directory generates the actual output plan.

### planner.c

**The query optimizer external interface.**

### planmain.c

**Routines to plan a single query.**

```c
* What's in a name, anyway?  The top-level entry point of the planner/
* optimizer is over in planner.c, not here as you might think from the
* file name.  But this is the main code for planning a basic join operation,
* shorn of features like subselects, inheritance, aggregates, grouping,
* and so on.  (Those are the things planner.c deals with.)
```

### initsplan.c

**Target list, qualification, joininfo initialization routines.**

### analyzejoins.c

**Routines for simplifying joins after initial query analysis.**

```c
* While we do a great deal of join simplification in prep/prepjointree.c,
* certain optimizations cannot be performed at that stage for lack of
* detailed information about the query.  The routines here are invoked
* after initsplan.c has done its work, and can do additional join removal
* and simplification steps based on the information extracted.  The penalty
* is that we have to work harder to clean up after ourselves when we modify
* the query, since the derived data structures have to be updated too.
```

### planagg.c

**Special planning for aggregate queries.**

```c
* This module tries to replace MIN/MAX aggregate functions by subqueries
* of the form
*     (SELECT col FROM tab
*      WHERE col IS NOT NULL AND existing-quals
*      ORDER BY col ASC/DESC
*      LIMIT 1)
* Given a suitable index on tab.col, this can be much faster than the
* generic scan-all-the-rows aggregation plan.  We can handle multiple
* MIN/MAX aggregates by generating multiple subqueries, and their
* orderings can be different.  However, if the query contains any
* non-optimizable aggregates, there's no point since we'll have to
* scan all the rows anyway.
```

### createplan.c

**Routines to create the desired plan for processing a query.**

```c
* Planning is complete, we just need to convert the selected
* Path into a Plan.
```

### setrefs.c

**Post-processing of a completed plan tree: fix references to subplan vars, compute regproc values for operators, etc.**

### subselect.c

**Planning routines for subselects.**

```c
* This module deals with SubLinks and CTEs, but not subquery RTEs (i.e.,
* not sub-SELECT-in-FROM cases).
```

## /path

### allpaths.c

**Routines to find possible search paths for processing a query.**

### clausesel.c

**Routines to compute clause selectivities.**

### costsize.c

**Routines to compute (and set) relation sizes and path costs.**

```c
* Path costs are measured in arbitrary units established by these basic
* parameters:
*
*  seq_page_cost     Cost of a sequential page fetch
*  random_page_cost   Cost of a non-sequential page fetch
*  cpu_tuple_cost    Cost of typical CPU time to process a tuple
*  cpu_index_tuple_cost  Cost of typical CPU time to process an index tuple
*  cpu_operator_cost  Cost of CPU time to execute an operator or function
*  parallel_tuple_cost Cost of CPU time to pass a tuple from worker to leader backend
*  parallel_setup_cost Cost of setting up shared memory for parallelism
*
* We expect that the kernel will typically do some amount of read-ahead
* optimization; this in conjunction with seek costs means that seq_page_cost
* is normally considerably less than random_page_cost.  (However, if the
* database is fully cached in RAM, it is reasonable to set them equal.)
*
* We also use a rough estimate "effective_cache_size" of the number of
* disk pages in Postgres + OS-level disk cache.  (We can't simply use
* NBuffers for this purpose because that would ignore the effects of
* the kernel's disk cache.)
*
* Obviously, taking constants for these values is an oversimplification,
* but it's tough enough to get any useful estimates even at this level of
* detail.  Note that all of these parameters are user-settable, in case
* the default values are drastically off for a particular platform.
*
* seq_page_cost and random_page_cost can also be overridden for an individual
* tablespace, in case some data is on a fast disk and other data is on a slow
* disk.  Per-tablespace overrides never apply to temporary work files such as
* an external sort or a materialize node that overflows work_mem.
*
* We compute two separate costs for each path:
*     total_cost: total estimated cost to fetch all tuples
*     startup_cost: cost that is expended before first tuple is fetched
* In some scenarios, such as when there is a LIMIT or we are implementing
* an EXISTS(...) sub-select, it is not necessary to fetch all tuples of the
* path's result.  A caller can estimate the cost of fetching a partial
* result by interpolating between startup_cost and total_cost.  In detail:
*     actual_cost = startup_cost +
*        (total_cost - startup_cost) * tuples_to_fetch / path->rows;
* Note that a base relation's rows count (and, by extension, plan_rows for
* plan nodes below the LIMIT node) are set without regard to any LIMIT, so
* that this equation works properly.  (Note: while path->rows is never zero
* for ordinary relations, it is zero for paths for provably-empty relations,
* so beware of division-by-zero.)  The LIMIT is applied as a top-level
* plan node.
*
* For largely historical reasons, most of the routines in this module use
* the passed result Path only to store their results (rows, startup_cost and
* total_cost) into.  All the input data they need is passed as separate
* parameters, even though much of it could be extracted from the Path.
* An exception is made for the cost_XXXjoin() routines, which expect all
* the other fields of the passed XXXPath to be filled in, and similarly
* cost_index() assumes the passed IndexPath is valid except for its output
* values.
```

### equivclass.c

**Routines for managing EquivalenceClasses.**

```c
* See src/backend/optimizer/README for discussion of EquivalenceClasses.
```

### indxpath.c

**Routines to determine which indexes are usable for scanning a given relation, and create Paths accordingly.**

### joinpath.c

**Routines to find all possible paths for processing a set of joins.**

### joinrels.c

**Routines to determine which relations should be joined.**

### pathkeys.c

**Utilities for matching and building path keys.**

### tidpath.c

**Routines to determine which TID conditions are usable for scanning a given relation, and create TidPaths and TidRangePaths accordingly.**

```c
* For TidPaths, we look for WHERE conditions of the form
* "CTID = pseudoconstant", which can be implemented by just fetching
* the tuple directly via heap_fetch().  We can also handle OR'd conditions
* such as (CTID = const1) OR (CTID = const2), as well as ScalarArrayOpExpr
* conditions of the form CTID = ANY(pseudoconstant_array).  In particular
* this allows
*     WHERE ctid IN (tid1, tid2, ...)
*
* As with indexscans, our definition of "pseudoconstant" is pretty liberal:
* we allow anything that doesn't involve a volatile function or a Var of
* the relation under consideration.  Vars belonging to other relations of
* the query are allowed, giving rise to parameterized TID scans.
*
* We also support "WHERE CURRENT OF cursor" conditions (CurrentOfExpr),
* which amount to "CTID = run-time-determined-TID".  These could in
* theory be translated to a simple comparison of CTID to the result of
* a function, but in practice it works better to keep the special node
* representation all the way through to execution.
*
* Additionally, TidRangePaths may be created for conditions of the form
* "CTID relop pseudoconstant", where relop is one of >,>=,<,<=, and
* AND-clauses composed of such conditions.
```

## /prep

## /util

## /geqo

