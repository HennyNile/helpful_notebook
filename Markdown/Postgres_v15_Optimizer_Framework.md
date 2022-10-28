# Framework of Postgres v15 Optimizer

**Postgres' source code has extremely clear paths compared with Hive and Spark. **

Code of Postgres' optimizer is located in **src/backend/optimizer** where there are 5 directories (prep, path, plan, geqo and utils). The following is [the official introduction of optimizer directory](https://github.com/postgres/postgres/tree/REL_15_STABLE/src/backend/optimizer).

```
These directories take the Query structure returned by the parser, and generate a plan used by the executor.  
/plan directory generates the actual output plan.
/path code generates all possible ways to join the tables.
/prep handles various preprocessing steps for special cases. 
/util is utility stuff.  
/geqo is the separate "genetic optimization" planner --- it does a semi-random search through the join tree space, rather than exhaustively considering all possible join trees.  (But each join considered by /geqo is given to /path to create paths for, so we consider all possible implementation paths for each specific join pair even in GEQO mode.)
```

Then let us dive into each subdirectories.

## /Plan

This directory generates the actual output plan.

