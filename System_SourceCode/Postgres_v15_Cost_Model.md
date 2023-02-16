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

## II. Cost Analysis

### 1.Scan

#### (1) SeqScan

```c
// disk cost
disk_run_cost = spc_seq_page_cost * baserel->pages; // spc_seq_page_cost is the cost of fetching a page

// cpu cost
startup_cost = qpqual_cost.startup // qpqual_cost is the cost of a where clause
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple; // cpu_tuple_cost = DEFAULT_CPU_TUPLE_COST
cpu_run_cost = cpu_per_tuple * baserel->tuples;

// tlist eval costs are paid per output row, not per tuple scanned
startup_cost += path->pathtarget->cost.startup;
cpu_run_cost += path->pathtarget->cost.per_tuple * path->rows;

/* Adjust costing for parallelism, if used. */
if (path->parallel_workers > 0)
{
    double		parallel_divisor = get_parallel_divisor(path);

    /* The CPU cost is divided among all the workers. */
    cpu_run_cost /= parallel_divisor;
    
    /*
		 * It may be possible to amortize some of the I/O cost, but probably
		 * not very much, because most operating systems already do aggressive
		 * prefetching.  For now, we assume that the disk run cost can't be
		 * amortized at all.
		 */

		/*
		 * In the case of a parallel plan, the row count needs to represent
		 * the number of tuples processed per worker.
		 */
    path->rows = clamp_row_est(path->rows / parallel_divisor);
}
```

#### (2) Index Scan

```c
// cost of scan index
amcostestimate = (amcostestimate_function) index->amcostestimate;
amcostestimate(root, path, loop_count,
               &indexStartupCost, &indexTotalCost,
               &indexSelectivity, &indexCorrelation,
               &index_pages);
path->indextotalcost = indexTotalCost;
path->indexselectivity = indexSelectivity;
startup_cost += indexStartupCost;
run_cost += indexTotalCost - indexStartupCost;
tuples_fetched = clamp_row_est(indexSelectivity * baserel->tuples);
get_tablespace_page_costs(baserel->reltablespace,
					      &spc_random_page_cost,
						  &spc_seq_page_cost);

// Estimate number of main-table pages fetched, and compute I/O cost.
pages_fetched = index_pages_fetched(tuples_fetched * loop_count,
                                    baserel->pages,
                                    (double) index->pages,
                                    root);
if (indexonly)
    pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));

rand_heap_pages = pages_fetched;

max_IO_cost = (pages_fetched * spc_random_page_cost) / loop_count;

pages_fetched = ceil(indexSelectivity * (double) baserel->pages);

pages_fetched = index_pages_fetched(pages_fetched * loop_count,
                                    baserel->pages,
                                    (double) index->pages,
                                    root);

if (indexonly)
    pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));

min_IO_cost = (pages_fetched * spc_random_page_cost) / loop_count;


/*
 * Now interpolate based on estimated index order correlation to get total
 * disk I/O cost for main table accesses.
 */
csquared = indexCorrelation * indexCorrelation;

run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost);

/*
 * Estimate CPU costs per tuple.
 *
 * What we want here is cpu_tuple_cost plus the evaluation costs of any
 * qual clauses that we have to evaluate as qpquals.
 */
path->path.rows = baserel->rows;
/* qpquals come from just the rel's restriction clauses */
qpquals = extract_nonindex_conditions(path->indexinfo->indrestrictinfo,
                                      path->indexclauses);

cost_qual_eval(&qpqual_cost, qpquals, root);
startup_cost += qpqual_cost.startup;
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;
cpu_run_cost += cpu_per_tuple * tuples_fetched;

/* tlist eval costs are paid per output row, not per tuple scanned */
startup_cost += path->path.pathtarget->cost.startup;
cpu_run_cost += path->path.pathtarget->cost.per_tuple * path->path.rows;

/* Adjust costing for parallelism, if used. */
if (path->path.parallel_workers > 0)
{
    double		parallel_divisor = get_parallel_divisor(&path->path);

    path->path.rows = clamp_row_est(path->path.rows / parallel_divisor);

    /* The CPU cost is divided among all the workers. */
    cpu_run_cost /= parallel_divisor;
}

run_cost += cpu_run_cost;

path->path.startup_cost = startup_cost;
path->path.total_cost = startup_cost + run_cost;
```

#### (3) Bitmap Scan

```c
// cost of scan index
pages_fetched = compute_bitmap_pages(root, baserel, bitmapqual,
                                     loop_count, &indexTotalCost,
                                     &tuples_fetched);

startup_cost += indexTotalCost;
T = (baserel->pages > 1) ? (double) baserel->pages : 1.0;

/* Fetch estimated page costs for tablespace containing table. */
get_tablespace_page_costs(baserel->reltablespace,
                          &spc_random_page_cost,
                          &spc_seq_page_cost);


// estimate I/O cost
if (pages_fetched >= 2.0)
    cost_per_page = spc_random_page_cost -
    (spc_random_page_cost - spc_seq_page_cost)
    * sqrt(pages_fetched / T);
else
    cost_per_page = spc_random_page_cost;

run_cost += pages_fetched * cost_per_page;

// estimate CPU cost
get_restriction_qual_cost(root, baserel, param_info, &qpqual_cost);

startup_cost += qpqual_cost.startup;
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;
cpu_run_cost = cpu_per_tuple * tuples_fetched;

/* Adjust costing for parallelism, if used. */
if (path->parallel_workers > 0)
{
    double		parallel_divisor = get_parallel_divisor(path);

    /* The CPU cost is divided among all the workers. */
    cpu_run_cost /= parallel_divisor;

    path->rows = clamp_row_est(path->rows / parallel_divisor);
}

run_cost += cpu_run_cost;

/* tlist eval costs are paid per output row, not per tuple scanned */
startup_cost += path->pathtarget->cost.startup;
run_cost += path->pathtarget->cost.per_tuple * path->rows;

path->startup_cost = startup_cost;
path->total_cost = startup_cost + run_cost;
```

### 2. Join

#### (1) Nestloop Join

##### Initial 

```c
// estimate costs to rescan the inner relation
cost_rescan(root, inner_path,
			&inner_rescan_start_cost,
			&inner_rescan_total_cost);

// cost of source data
startup_cost += outer_path->startup_cost + inner_path->startup_cost;
run_cost += outer_path->total_cost - outer_path->startup_cost;
if (outer_path_rows > 1)
    run_cost += (outer_path_rows - 1) * inner_rescan_start_cost;

inner_run_cost = inner_path->total_cost - inner_path->startup_cost;
inner_rescan_run_cost = inner_rescan_total_cost - inner_rescan_start_cost;

if (jointype == JOIN_SEMI || jointype == JOIN_ANTI ||
    extra->inner_unique)
{
    workspace->inner_run_cost = inner_run_cost;
    workspace->inner_rescan_run_cost = inner_rescan_run_cost;
}
else
{
    /* Normal case; we'll scan whole input rel for each outer row */
    run_cost += inner_run_cost;
    if (outer_path_rows > 1)
        run_cost += (outer_path_rows - 1) * inner_rescan_run_cost;
}

// CPU cost left for later 
```

##### Final

```c
// CPU cost
if (path->jointype == JOIN_SEMI || path->jointype == JOIN_ANTI ||
    extra->inner_unique)
{
    /*
		 * With a SEMI or ANTI join, or if the innerrel is known unique, the
		 * executor will stop after the first match.
		 */
    Cost		inner_run_cost = workspace->inner_run_cost;
    Cost		inner_rescan_run_cost = workspace->inner_rescan_run_cost;
    double		outer_matched_rows;
    double		outer_unmatched_rows;
    Selectivity inner_scan_frac;

    /*
		 * For an outer-rel row that has at least one match, we can expect the
		 * inner scan to stop after a fraction 1/(match_count+1) of the inner
		 * rows, if the matches are evenly distributed.  Since they probably
		 * aren't quite evenly distributed, we apply a fuzz factor of 2.0 to
		 * that fraction.  (If we used a larger fuzz factor, we'd have to
		 * clamp inner_scan_frac to at most 1.0; but since match_count is at
		 * least 1, no such clamp is needed now.)
		 */
    outer_matched_rows = rint(outer_path_rows * extra->semifactors.outer_match_frac);
    outer_unmatched_rows = outer_path_rows - outer_matched_rows;
    inner_scan_frac = 2.0 / (extra->semifactors.match_count + 1.0);

    /*
		 * Compute number of tuples processed (not number emitted!).  First,
		 * account for successfully-matched outer rows.
		 */
    ntuples = outer_matched_rows * inner_path_rows * inner_scan_frac;

    /*
		 * Now we need to estimate the actual costs of scanning the inner
		 * relation, which may be quite a bit less than N times inner_run_cost
		 * due to early scan stops.  We consider two cases.  If the inner path
		 * is an indexscan using all the joinquals as indexquals, then an
		 * unmatched outer row results in an indexscan returning no rows,
		 * which is probably quite cheap.  Otherwise, the executor will have
		 * to scan the whole inner rel for an unmatched row; not so cheap.
		 */
    if (has_indexed_join_quals(path))
    {
        /*
			 * Successfully-matched outer rows will only require scanning
			 * inner_scan_frac of the inner relation.  In this case, we don't
			 * need to charge the full inner_run_cost even when that's more
			 * than inner_rescan_run_cost, because we can assume that none of
			 * the inner scans ever scan the whole inner relation.  So it's
			 * okay to assume that all the inner scan executions can be
			 * fractions of the full cost, even if materialization is reducing
			 * the rescan cost.  At this writing, it's impossible to get here
			 * for a materialized inner scan, so inner_run_cost and
			 * inner_rescan_run_cost will be the same anyway; but just in
			 * case, use inner_run_cost for the first matched tuple and
			 * inner_rescan_run_cost for additional ones.
			 */
        run_cost += inner_run_cost * inner_scan_frac;
        if (outer_matched_rows > 1)
            run_cost += (outer_matched_rows - 1) * inner_rescan_run_cost * inner_scan_frac;

        /*
			 * Add the cost of inner-scan executions for unmatched outer rows.
			 * We estimate this as the same cost as returning the first tuple
			 * of a nonempty scan.  We consider that these are all rescans,
			 * since we used inner_run_cost once already.
			 */
        run_cost += outer_unmatched_rows *
            inner_rescan_run_cost / inner_path_rows;

        /*
			 * We won't be evaluating any quals at all for unmatched rows, so
			 * don't add them to ntuples.
			 */
    }
    else
    {
        /*
			 * Here, a complicating factor is that rescans may be cheaper than
			 * first scans.  If we never scan all the way to the end of the
			 * inner rel, it might be (depending on the plan type) that we'd
			 * never pay the whole inner first-scan run cost.  However it is
			 * difficult to estimate whether that will happen (and it could
			 * not happen if there are any unmatched outer rows!), so be
			 * conservative and always charge the whole first-scan cost once.
			 * We consider this charge to correspond to the first unmatched
			 * outer row, unless there isn't one in our estimate, in which
			 * case blame it on the first matched row.
			 */

        /* First, count all unmatched join tuples as being processed */
        ntuples += outer_unmatched_rows * inner_path_rows;

        /* Now add the forced full scan, and decrement appropriate count */
        run_cost += inner_run_cost;
        if (outer_unmatched_rows >= 1)
            outer_unmatched_rows -= 1;
        else
            outer_matched_rows -= 1;

        /* Add inner run cost for additional outer tuples having matches */
        if (outer_matched_rows > 0)
            run_cost += outer_matched_rows * inner_rescan_run_cost * inner_scan_frac;

        /* Add inner run cost for additional unmatched outer tuples */
        if (outer_unmatched_rows > 0)
            run_cost += outer_unmatched_rows * inner_rescan_run_cost;
    }
}
else
{
    /* Normal-case source costs were included in preliminary estimate */

    /* Compute number of tuples processed (not number emitted!) */
    ntuples = outer_path_rows * inner_path_rows;
}
```

#### (2) Merge Join

##### Initial

```c
/* Get the selectivity with caching */
cache = cached_scansel(root, firstclause, opathkey);

if (bms_is_subset(firstclause->left_relids,
                  outer_path->parent->relids))
{
    /* left side of clause is outer */
    outerstartsel = cache->leftstartsel;
    outerendsel = cache->leftendsel;
    innerstartsel = cache->rightstartsel;
    innerendsel = cache->rightendsel;
}
else
{
    /* left side of clause is inner */
    outerstartsel = cache->rightstartsel;
    outerendsel = cache->rightendsel;
    innerstartsel = cache->leftstartsel;
    innerendsel = cache->leftendsel;
}

/*
 * Convert selectivities to row counts.  We force outer_rows and
 * inner_rows to be at least 1, but the skip_rows estimates can be zero.
 */
outer_skip_rows = rint(outer_path_rows * outerstartsel);
inner_skip_rows = rint(inner_path_rows * innerstartsel);
outer_rows = clamp_row_est(outer_path_rows * outerendsel);
inner_rows = clamp_row_est(inner_path_rows * innerendsel);

/*
 * Readjust scan selectivities to account for above rounding.  This is
 * normally an insignificant effect, but when there are only a few rows in
 * the inputs, failing to do this makes for a large percentage error.
 */
outerstartsel = outer_skip_rows / outer_path_rows;
innerstartsel = inner_skip_rows / inner_path_rows;
outerendsel = outer_rows / outer_path_rows;
innerendsel = inner_rows / inner_path_rows;

/* cost of source data */

if (outersortkeys)			/* do we need to sort outer? */
{
    cost_sort(&sort_path,
              root,
              outersortkeys,
              outer_path->total_cost,
              outer_path_rows,
              outer_path->pathtarget->width,
              0.0,
              work_mem,
              -1.0);
    startup_cost += sort_path.startup_cost;
    startup_cost += (sort_path.total_cost - sort_path.startup_cost)
        * outerstartsel;
    run_cost += (sort_path.total_cost - sort_path.startup_cost)
        * (outerendsel - outerstartsel);
}
else
{
    startup_cost += outer_path->startup_cost;
    startup_cost += (outer_path->total_cost - outer_path->startup_cost)
        * outerstartsel;
    run_cost += (outer_path->total_cost - outer_path->startup_cost)
        * (outerendsel - outerstartsel);
}

if (innersortkeys)			/* do we need to sort inner? */
{
    cost_sort(&sort_path,
              root,
              innersortkeys,
              inner_path->total_cost,
              inner_path_rows,
              inner_path->pathtarget->width,
              0.0,
              work_mem,
              -1.0);
    startup_cost += sort_path.startup_cost;
    startup_cost += (sort_path.total_cost - sort_path.startup_cost)
        * innerstartsel;
    inner_run_cost = (sort_path.total_cost - sort_path.startup_cost)
        * (innerendsel - innerstartsel);
}
else
{
    startup_cost += inner_path->startup_cost;
    startup_cost += (inner_path->total_cost - inner_path->startup_cost)
        * innerstartsel;
    inner_run_cost = (inner_path->total_cost - inner_path->startup_cost)
        * (innerendsel - innerstartsel);
}

/* CPU costs left for later */
```

##### Final

```c
/*
 * Compute cost of the mergequals and qpquals (other restriction clauses)
 * separately.
 */
cost_qual_eval(&merge_qual_cost, mergeclauses, root);
cost_qual_eval(&qp_qual_cost, path->jpath.joinrestrictinfo, root);
qp_qual_cost.startup -= merge_qual_cost.startup;
qp_qual_cost.per_tuple -= merge_qual_cost.per_tuple;

/*
 * Get approx # tuples passing the mergequals.  We use approx_tuple_count
 * here because we need an estimate done with JOIN_INNER semantics.
 */
mergejointuples = approx_tuple_count(root, &path->jpath, mergeclauses);

if (IsA(outer_path, UniquePath) || path->skip_mark_restore)
    rescannedtuples = 0;
else
{
    rescannedtuples = mergejointuples - inner_path_rows;
    /* Must clamp because of possible underestimate */
    if (rescannedtuples < 0)
        rescannedtuples = 0;
}

/*
 * We'll inflate various costs this much to account for rescanning.  Note
 * that this is to be multiplied by something involving inner_rows, or
 * another number related to the portion of the inner rel we'll scan.
 */
rescanratio = 1.0 + (rescannedtuples / inner_rows);

/*
 * Decide whether we want to materialize the inner input to shield it from
 * mark/restore and performing re-fetches.  Our cost model for regular
 * re-fetches is that a re-fetch costs the same as an original fetch,
 * which is probably an overestimate; but on the other hand we ignore the
 * bookkeeping costs of mark/restore.  Not clear if it's worth developing
 * a more refined model.  So we just need to inflate the inner run cost by
 * rescanratio.
 */
bare_inner_cost = inner_run_cost * rescanratio;

/*
 * When we interpose a Material node the re-fetch cost is assumed to be
 * just cpu_operator_cost per tuple, independently of the underlying
 * plan's cost; and we charge an extra cpu_operator_cost per original
 * fetch as well.  Note that we're assuming the materialize node will
 * never spill to disk, since it only has to remember tuples back to the
 * last mark.  (If there are a huge number of duplicates, our other cost
 * factors will make the path so expensive that it probably won't get
 * chosen anyway.)	So we don't use cost_rescan here.
 *
 * Note: keep this estimate in sync with create_mergejoin_plan's labeling
 * of the generated Material node.
 */
mat_inner_cost = inner_run_cost +
    cpu_operator_cost * inner_rows * rescanratio;

if (path->skip_mark_restore)
    path->materialize_inner = false;
else if (enable_material && mat_inner_cost < bare_inner_cost)
    path->materialize_inner = true;
else if (innersortkeys == NIL &&
     	!ExecSupportsMarkRestore(inner_path))
    path->materialize_inner = true;

/* Charge the right incremental cost for the chosen case */
if (path->materialize_inner)
    run_cost += mat_inner_cost;
else
    run_cost += bare_inner_cost;

/* CPU costs */

/*
 * The number of tuple comparisons needed is approximately number of outer
 * rows plus number of inner rows plus number of rescanned tuples (can we
 * refine this?).  At each one, we need to evaluate the mergejoin quals.
 */
startup_cost += merge_qual_cost.startup;
startup_cost += merge_qual_cost.per_tuple *
    (outer_skip_rows + inner_skip_rows * rescanratio);
run_cost += merge_qual_cost.per_tuple *
    ((outer_rows - outer_skip_rows) +
     (inner_rows - inner_skip_rows) * rescanratio);

/*
 * For each tuple that gets through the mergejoin proper, we charge
 * cpu_tuple_cost plus the cost of evaluating additional restriction
 * clauses that are to be applied at the join.  (This is pessimistic since
 * not all of the quals may get evaluated at each tuple.)
 *
 * Note: we could adjust for SEMI/ANTI joins skipping some qual
 * evaluations here, but it's probably not worth the trouble.
 */
startup_cost += qp_qual_cost.startup;
cpu_per_tuple = cpu_tuple_cost + qp_qual_cost.per_tuple;
run_cost += cpu_per_tuple * mergejointuples;

/* tlist eval costs are paid per output row, not per tuple scanned */
startup_cost += path->jpath.path.pathtarget->cost.startup;
run_cost += path->jpath.path.pathtarget->cost.per_tuple * path->jpath.path.rows;

path->jpath.path.startup_cost = startup_cost;
path->jpath.path.total_cost = startup_cost + run_cost;
```

#### (3) Hash Join

##### Initial

```c
/*
 * Cost of computing hash function: must do it once per input tuple. We
 * charge one cpu_operator_cost for each column's hash function.  Also,
 * tack on one cpu_tuple_cost per inner row, to model the costs of
 * inserting the row into the hashtable.
 *
 * XXX when a hashclause is more complex than a single operator, we really
 * should charge the extra eval costs of the left or right side, as
 * appropriate, here.  This seems more work than it's worth at the moment.
 */
startup_cost += (cpu_operator_cost * num_hashclauses + cpu_tuple_cost)
    * inner_path_rows;
run_cost += cpu_operator_cost * num_hashclauses * outer_path_rows;

/*
 * Get hash table size that executor would use for inner relation.
 *
 * XXX for the moment, always assume that skew optimization will be
 * performed.  As long as SKEW_HASH_MEM_PERCENT is small, it's not worth
 * trying to determine that for sure.
 *
 * XXX at some point it might be interesting to try to account for skew
 * optimization in the cost estimate, but for now, we don't.
 */
ExecChooseHashTableSize(inner_path_rows_total,
                        inner_path->pathtarget->width,
                        true,	/* useskew */
                        parallel_hash,	/* try_combined_hash_mem */
                        outer_path->parallel_workers,
                        &space_allowed,
                        &numbuckets,
                        &numbatches,
                        &num_skew_mcvs);

/*
 * If inner relation is too big then we will need to "batch" the join,
 * which implies writing and reading most of the tuples to disk an extra
 * time.  Charge seq_page_cost per page, since the I/O should be nice and
 * sequential.  Writing the inner rel counts as startup cost, all the rest
 * as run cost.
 */
if (numbatches > 1)
{
    double		outerpages = page_size(outer_path_rows,
                                       outer_path->pathtarget->width);
    double		innerpages = page_size(inner_path_rows,
                                       inner_path->pathtarget->width);

    startup_cost += seq_page_cost * innerpages;
    run_cost += seq_page_cost * (innerpages + 2 * outerpages);
}

/* CPU costs left for later */
```

##### Final

```c
/* mark the path with estimated # of batches */
path->num_batches = numbatches;

/* store the total number of tuples (sum of partial row estimates) */
path->inner_rows_total = inner_path_rows_total;

/* and compute the number of "virtual" buckets in the whole join */
virtualbuckets = (double) numbuckets * (double) numbatches;

if (IsA(inner_path, UniquePath))
{
    innerbucketsize = 1.0 / virtualbuckets;
    innermcvfreq = 0.0;
}
else
{
    innerbucketsize = 1.0;
    innermcvfreq = 1.0;
    foreach(hcl, hashclauses)
    {
        thisbucketsize = ...;
        thismcvfreq = ...;
        if (innerbucketsize > thisbucketsize)
				innerbucketsize = thisbucketsize;
        if (innermcvfreq > thismcvfreq)
            innermcvfreq = thismcvfreq;
    }
}

/*
 * Compute cost of the hashquals and qpquals (other restriction clauses)
 * separately.
 */
cost_qual_eval(&hash_qual_cost, hashclauses, root);
cost_qual_eval(&qp_qual_cost, path->jpath.joinrestrictinfo, root);
qp_qual_cost.startup -= hash_qual_cost.startup;
qp_qual_cost.per_tuple -= hash_qual_cost.per_tuple;

/* CPU costs */

if (path->jpath.jointype == JOIN_SEMI ||
		path->jpath.jointype == JOIN_ANTI ||
		extra->inner_unique)
{
    double		outer_matched_rows;
    Selectivity inner_scan_frac;
    
    outer_matched_rows = rint(outer_path_rows * extra->semifactors.outer_match_frac);
    inner_scan_frac = 2.0 / (extra->semifactors.match_count + 1.0);

    startup_cost += hash_qual_cost.startup;
    run_cost += hash_qual_cost.per_tuple * outer_matched_rows *
        clamp_row_est(inner_path_rows * innerbucketsize * inner_scan_frac) * 0.5;
    
    run_cost += hash_qual_cost.per_tuple *
        (outer_path_rows - outer_matched_rows) *
        clamp_row_est(inner_path_rows / virtualbuckets) * 0.05;

    /* Get # of tuples that will pass the basic join */
    if (path->jpath.jointype == JOIN_ANTI)
        hashjointuples = outer_path_rows - outer_matched_rows;
    else
        hashjointuples = outer_matched_rows;
}
else
{
    /*
     * The number of tuple comparisons needed is the number of outer
     * tuples times the typical number of tuples in a hash bucket, which
     * is the inner relation size times its bucketsize fraction.  At each
     * one, we need to evaluate the hashjoin quals.  But actually,
     * charging the full qual eval cost at each tuple is pessimistic,
     * since we don't evaluate the quals unless the hash values match
     * exactly.  For lack of a better idea, halve the cost estimate to
     * allow for that.
     */
    startup_cost += hash_qual_cost.startup;
    run_cost += hash_qual_cost.per_tuple * outer_path_rows *
        clamp_row_est(inner_path_rows * innerbucketsize) * 0.5;

    /*
     * Get approx # tuples passing the hashquals.  We use
     * approx_tuple_count here because we need an estimate done with
     * JOIN_INNER semantics.
     */
    hashjointuples = approx_tuple_count(root, &path->jpath, hashclauses);
}

/*
 * For each tuple that gets through the hashjoin proper, we charge
 * cpu_tuple_cost plus the cost of evaluating additional restriction
 * clauses that are to be applied at the join.  (This is pessimistic since
 * not all of the quals may get evaluated at each tuple.)
 */
startup_cost += qp_qual_cost.startup;
cpu_per_tuple = cpu_tuple_cost + qp_qual_cost.per_tuple;
run_cost += cpu_per_tuple * hashjointuples;

/* tlist eval costs are paid per output row, not per tuple scanned */
startup_cost += path->jpath.path.pathtarget->cost.startup;
run_cost += path->jpath.path.pathtarget->cost.per_tuple * path->jpath.path.rows;

path->jpath.path.startup_cost = startup_cost;
path->jpath.path.total_cost = startup_cost + run_cost;
```

## III. Cost Analysis Experiments

### 0. JOB Benchmark Info

 

|      Table      | Row Number |                 Indexed Columns                  | Unindexed Column |
| :-------------: | :--------: | :----------------------------------------------: | :--------------: |
|    cast_info    |  36231584  | id, movie_id, person_id, person_role_id, role_id |     nr_order     |
|   movie_info    |  14803594  |            id, movie_id, info_type_id            |       info       |
|  movie_keyword  |  4523930   |             id, movie_id, keyword_id             |                  |
|      name       |  4167151   |                        id                        |       name       |
|    char_name    |  3140292   |                        id                        |       name       |
|   person_info   |  2966063   |           id, person_id, info_type_id            |       info       |
| movie_companies |  2609129   |    id, company_id, movie_id, company_type_id     |       note       |
|      title      |  2528527   |              id, kind_id, movie_id               |      title       |
| movie_info_idx  |  1380035   |            id, movie_id, info_type_id            |       info       |
|    aka_name     |   901343   |                  id, person_id                   |       name       |
|    aka_title    |   361472   |                   id, kind_id                    |     movie_id     |
|  company_name   |   234997   |                        id                        |       name       |
|  complete_cast  |   135086   |                   id, movie_id                   |    subject_id    |
|     keyword     |   134170   |                        id                        |     keyword      |
|   movie_link    |   29997    |   id, movie_id, linked_movie_id, link_type_id    |                  |
|    info_type    |    113     |                        id                        |       info       |
|    link_type    |     18     |                        id                        |       link       |
|    role_type    |     12     |                        id                        |       role       |
|    kind_type    |     7      |                        id                        |       kind       |
| comp_cast_type  |     4      |                        id                        |       kind       |
|  company_type   |     4      |                        id                        |       kind       |



### 1. Scan

