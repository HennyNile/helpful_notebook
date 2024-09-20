# Setup Dataset used in Benchmarks

This file includes the setup of **JOB (Join Order Benchmark)**, **SSB (Star Schema Benchmark)**, **TPC-H** now.

## I. Dumped IMDB Dataset

Dataset Resource: https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT

Main Reference: https://www.postgresql.org/docs/7.1/app-pgdump.html, https://www.postgresql.org/docs/current/app-pgrestore.html

### 1. Get the Source

Download from https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT.

### 2. Load the Data 

The data is dumped from postgres database with command 

```bash
pg_dump -Fc
```

Then we use pg_restore to restore data

```bash
su - postgres_15_sc
pg_restore -h localhost -p 5431 -U postgres_15_sc -C -d postgres /tmp/imdb_pg11
```

## II. Neurocard IMDB Dataset

**Main Reference**: https://github.com/neurocard/neurocard/blob/master/neurocard/scripts/download_imdb.sh

### 1. Download the data

```
wget -c http://homepages.cwi.nl/~boncz/job/imdb.tgz && tar -xvzf imdb.tgz
```

### 2. Load data into PG

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os

if __name__ == '__main__':
    csv_files = os.listdir()
    csv_files = [file for file in csv_files if file.endswith('.csv')]
    csv_files.sort()
    for file in csv_files:
        print(f"COPY {file[:-4]} FROM '{os.getcwd()}/{file}' CSV ESCAPE '\\';")
```

## III. SSB

main reference: 

https://clickhouse.com/docs/zh/getting-started/example-datasets/star-schema/

https://quickstep.cs.wisc.edu/benchmark-scripts/

### 1. Get the Source

#### a. Clone ssb-dbgen.git

```bash
git clone https://github.com/vadimtk/ssb-dbgen.git
```

#### b. Modify dss.h

```c
// change filed seperator from , to |
// initial
define SEPARATOR ','
    
// modified
define SEPARATOR '|'
```

#### c. Modify print.c

```c
// remove the double quotion marks in the string fields
// initial
fprintf(target, "\"%s\"", (char *)data);

// modified
fprintf(target, "%s", (char *)data);
```

#### d. Generate Dataset

```bash
cd ssb-dbgen
make

# generate dataset
# (customer.tbl)
dbgen -s 1 -T c

#(part.tbl)
dbgen -s 1 -T p

#(supplier.tbl)
dbgen -s 1 -T s

#(date.tbl)
dbgen -s 1 -T d

#(fact table lineorder.tbl)
dbgen -s 1 -T l

#(for all SSBM tables)
dbgen -s 1 -T a

To generate the refresh (insert/delete) data set:
(create delete.[1-4] and lineorder.tbl.u[1-4] with refreshing fact 0.05%)
dbgen -s 1 -r 5 -U 4

   where "-r 5" specifies refreshin fact n/10000
         "-U 4" specifies 4 segments for deletes and inserts
```

#### e. Modify .tbl files

```bash
# use vim to remove last character of each line, take part.tbl as an example
vim part.tbl 
:%normal $x

# remove all lines containing ", these lines are repeated.
:g/pattern/d

#	: 让你进入命令行模式
#	% 是表示整个文件的范围
#	normal 说我们正在运行普通模式命令
#	$x 删除行中的最后一个字符	

# replace split character , to |
# :%s/","/"|"/g

# remove all "
# :%s/"/g
```

#### f. Rename date.tbl to dates.tbl

### 2. Load the Data

Download the scripts for postgres from https://quickstep.cs.wisc.edu/benchmark-scripts/.

Use the script ```load.sh``` to load .tbl dataset into postgres.

Some preparation is needed.

#### a. Create a database for SSB

```postgresql
# in psql command line
psql> create database ssb_1
```

#### b. Modify config.sh

```bash
# You need to copy the dataset to a dir which the owner of postgresql could access
SSB_TABLE_FILES_DIR="/home/postgres_15_sc/SSB/dataset"

POSTGRES_EXEC="/usr/local/pgsql/bin/psql"

# the database name you create in last step
POSTGRES_DB_NAME="ssb_1"
```

#### c. Modify load.sh

```bash
# Add parameter for POSTGRES_EXEC
# initial
$POSTGRES_EXEC -d $POSTGRES_DB_NAME -f ./create.sql

# modified
$POSTGRES_EXEC -p 5431 -U postgres_15_sc -d $POSTGRES_DB_NAME -f ./create.sql

# initial
table_list=(
    part
    supplier
    customer
    ddate
    lineorder
)

# modified
table_list=(
    part
    supplier
    customer
    dates
    lineorder
)

# initial
ALTER TABLE lineorder
ADD FOREIGN KEY(lo_orderdate) REFERENCES ddate(d_datekey);

ALTER TABLE lineorder
ADD FOREIGN KEY(lo_commitdate) REFERENCES ddate(d_datekey);

# modified
ALTER TABLE lineorder
ADD FOREIGN KEY(lo_orderdate) REFERENCES dates(d_datekey);

ALTER TABLE lineorder
ADD FOREIGN KEY(lo_commitdate) REFERENCES dates(d_datekey);
```

#### d. Modify create.sql

```sql
# initial
DROP TABLE IF EXISTS ddate CASCADE;

# modified
DROP TABLE IF EXISTS dates CASCADE;

# initial
CREATE TABLE ddate 
    
# modified
CREATE TABLE dates 
    
# initial
lo_orderdate INT NOT NULL,
lo_commitdate INT NOT NULL,

# modified
lo_orderdate DATE NOT NULL,
lo_commitdate DATE NOT NULL,

# initial
d_datekey INT NOT NULL,

# modified
d_datekey DATE NOT NULL,
```

#### e. Copy dataset to the $SSB_TABLE_FILES_DIR

#### f. Run Command of Loading Data

```bash
sh load.sh
```

## IV. TPC-H

main reference: 

https://github.com/electrum/tpch-dbgen

https://github.com/tvondra/pg_tpch

toolkit website: 

https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp

### 1. Configure DBGen

```bash
# 1. dowoload dbgen from https://www.tpc.org/tpch/

# 2. modify dbgen code
cd dbgen
cp Makefile.suite Makefile
vim Makefile
	CC=gcc
	DataBase=POSTGRES
	MACHINE=LINUX
	WORKLOAD=TPCH

vim tpcd.h
# add following
#ifdef POSTGRES
#define GEN_QUERY_PLAN  "EXPLAIN"
#define START_TRAN      "BEGIN"
#define END_TRAN        "END"
#define SET_OUTPUT      ""
#define SET_ROWCOUNT    "limit %d\n"
#define SET_DBASE       "\\c %s\n"
#endif

make

# if meet error "malloc.h not included", replace all "malloc.h" with stdlib.h
```

### 2. Generate Dataset

```bash
#!/bin/bash

scale_factor=10

rm -rf results/dataset/sf${scale_factor}
mkdir -p results/dataset/sf${scale_factor}

cd dbgen

./dbgen -s $scale_factor

for i in `ls *.tbl`; 
do
    sed 's/|$//' $i > ../results/dataset/sf${scale_factor}/${i/tbl/csv}
done

cd ../results/dataset/sf${scale_factor}

table_names=('nation' 'region' 'part' 'supplier' 'partsupp' 'customer' 'orders' 'lineitem')

for i in "${table_names[@]}";
do
    echo copy $i from \'${PWD}/$i.csv\' delimiter \'\|\' csv\;;
done
```

### 3. Load Data into PG

```bash
# 1. create database
(SQL) create database tpch_[scale factor]

# 2. use sql in reference github repository (second reference) create tables
\i tpch-create.sql

# 3. load data into pg
(SQL) copy tblname from 'tbl_csv_path' delimiter '|' csv;

# 4. use sql in reference github repository (second reference) create primary key, foreign key and addition index 
\i tpch-pkeys.sql
\i tpch-alter.sql
\i tpch-index.sql
```

### 4. Generate Queries

```bash
selected_templates=(3 5 7 8 9 10)
template_query_num=200

rm -rf results/queries

mkdir -p results/queries

cd dbgen

for i in "${selected_templates[@]}"
for j in $(seq 1 $template_query_num)
do
    teplate_number=$(printf "%02d" $i)
    query_number=$(printf "%03d" $j)
    # echo Running query ${teplate_number}_${query_number}.sql...
    DSS_QUERY=queries ./qgen -r $(($RANDOM*$RANDOM)) $i >> ../results/queries/${teplate_number}_${query_number}.sql
done
```

## V. TPC-DS

main reference:

https://github.com/gregrahn/tpcds-kit/blob/master/tools/s_purchase.c

https://github.com/celuk/tpcds-postgres

toolkit website: 

https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp

### 1. Configure DBGen

```bash
sudo apt-get install gcc make flex bison byacc git

vim makefile

LINUX_CFLAGS = -g -Wall → -O3 -Wall -fcommon # force the compiler to accept repeated definition

make OS=LINUX
```

### 2. Generate Dataset

```bash
#!/bin/bash

scale_factor=10

rm -rf results/datasets/sf${scale_factor}
mkdir -p results/datasets/sf${scale_factor}

cd tools

./dsdgen -SCALE 10 -DIR ../results/datasets/sf10/

cd ../results/datasets/sf${scale_factor}

for i in `ls *.dat`; 
do
    sed 's/|$//' $i > $i.csv
done

for i in `ls *.csv`;
do
    table_name=$(echo "$i" | cut -d'.' -f1)
    echo copy $table_name from \'${PWD}/$i\' delimiter \'\|\' csv\;
done
```

### 3. Load Data into PG

```bash
# 1. create database
(SQL) create database tpcds_[scale factor]

# 2. create tables
\i tools/tpcds.sql
\i tools/tpcds_source.sql

# 3. load data into pg
(SQL) copy tblname from 'tbl_csv_path' delimiter '|' csv;
# be careful that the data of customer table may include invalid character which will lead failed import.

# 4. use sql in reference github repository (second reference) create primary key, foreign key and addition index 
\i tpcds_ri.sql

ANALYZE VERBOSE
```

### 4. Generate Queries

```bash
# modify netezza.tpl
add "define _END=""

# modify template for postgres
n days → interval 'n day'
```

```bash
selected_templates=(21 46 67 89 53 79 68 61 96 7 19 25 86 91 43 27 36 62 37 98 85 17 3 29 22 15 52 50 42 12 20 82 84 55 26 18 99)
template_query_num=30

rm -rf results/queries
mkdir -p results/queries

cd tools

for i in "${selected_templates[@]}"
for j in $(seq 1 $template_query_num)
do
    template_number=$(printf "%02d" $i)
    query_number=$(printf "%03d" $j)
    ./dsqgen -DIRECTORY ../query_templates_modified -INPUT ../query_templates_modified/templates.lst -QUALIFY Y -SCALE 10 -OUTPUT_DIR ../results/queries/ -DIALECT netezza -TEMPLATE query$i.tpl -RNGSEED $(($RANDOM*$RANDOM))
    cp ../results/queries/query_0.sql ../results/queries/${template_number}_${query_number}.sql
done

if test -f "../results/queries/query_0.sql"; then
    rm ../results/queries/query_0.sql
fi

if test -f "../results/queries/query_1.sql"; then
    rm ../results/queries/query_1.sql
fi
```

```python
# code to repeatedly invoke the upper shell commands to generate checked queries and generate final workloads

import os
import subprocess

initial_queries_dir = 'results/queries/'
checked_queries_dir = 'results/checked_queries/'
template_query_num = 30
Benchmark = 'TPCDS'

def load_queries(queries_dir):
    queries = {}
    for filename in os.listdir(queries_dir):
        template_number = filename.split('_')[0]
        if template_number not in queries:
            queries[template_number] = []
        with open(queries_dir + filename, 'r') as file:
            query = '\n'.join(file.read().split('\n')[1:])
            queries[template_number].append(query)
    return queries 

def save_queries(dir, queries):
    for template_number, template_queries in queries.items():
        for i in range(len(template_queries)):
            query = '\n' + template_queries[i]
            with open(dir + template_number + '_' + '{:03d}'.format(i+1) + '.sql', 'w') as file:
                file.write(query)

def generate_queries():
    subprocess.run(['zsh', 'gen_queries.sh', '> /dev/null', '2>&1'])

def check_queries():
    os.makedirs(checked_queries_dir, exist_ok=True)

    initial_queries = load_queries(initial_queries_dir)
    checked_queries = load_queries(checked_queries_dir)

    print('Initial checked queries number:')
    queries_num = 0
    for template_number, queries in checked_queries.items():
        print(f'template {template_number}: {len(queries)}')
        queries_num += len(queries)

    print(f'All queries number: {queries_num}')

    checked_queries_set = {}
    for template_number, queries in checked_queries.items():
        checked_queries_set[template_number] = set(queries)

    for template_number, queries in initial_queries.items():
        if template_number not in checked_queries:
            checked_queries[template_number] = []
            checked_queries_set[template_number] = set()
        elif len(checked_queries[template_number]) >= template_query_num:
            continue
        for query in queries:
            if query not in checked_queries_set[template_number]:
                checked_queries[template_number].append(query)
                checked_queries_set[template_number].add(query)
    
    print('New checked queries number:')
    queries_num = 0
    for template_number, queries in checked_queries.items():
        print(f'template {template_number}: {len(queries)}')
        queries_num += len(queries)

    print(f'All queries number: {queries_num}')

    save_queries(checked_queries_dir, checked_queries)

if __name__ == '__main__':
    task = 1 # 0 - generate new queries, 1 - generate final queries
    if task == 0:
        os.makedirs(checked_queries_dir, exist_ok=True)
        check_queries()
        checked_queries = load_queries(checked_queries_dir)

        all_satisfied = False
        iteration_num = 0
        while not all_satisfied:
            print(f'Iteration {iteration_num}')
            iteration_num += 1
            all_satisfied = True
            for template_number, queries in checked_queries.items():
                if len(queries) < template_query_num:
                    all_satisfied = False
            
            if not all_satisfied:
                generate_queries()
                check_queries()
                # break
    elif task == 1:
        all_queries_num, train_queries_num, test_queries_num = 0, 0, 0
        check_queries = load_queries(checked_queries_dir)
        train_queries, test_queries = {}, {}
        for template_number, queries in check_queries.items():
            if len(queries) > template_query_num:
                queries = queries[:template_query_num]
            template_train_queries_num = int(len(queries)/7)
            if template_train_queries_num < 1:
                template_train_queries_num = 1
            template_train_queries = queries[:template_train_queries_num]
            template_test_queries = queries[template_train_queries_num:]
            all_queries_num += len(queries)
            train_queries_num += len(template_train_queries)
            test_queries_num += len(template_test_queries)
            train_queries[template_number] = template_train_queries
            test_queries[template_number] = template_test_queries
        print(f'All queries number: {all_queries_num}, train queries number: {train_queries_num}, test queries number: {test_queries_num}')
        os.makedirs(f'results/{Benchmark}/', exist_ok=True)
        os.makedirs(f'results/{Benchmark}-sample/', exist_ok=True)
        save_queries(f'results/{Benchmark}/', train_queries)
        save_queries(f'results/{Benchmark}-sample/', test_queries)
```



## VI. STATS

source url: https://github.com/Nathaniel-Han/End-to-End-CardEst-Benchmark

### 1. Load Data into PG

The csv data file is in the dir ``datasets/stats_simplified``.

```sql
create database stats;
\c stats;
\i datasets/stats_simplified/stats.sql
\i scripts/sql/stats_load.sql
\i scripts/sql/stats_index.sql
```

### 2. Workloads

The workload queries locate at ``workloads``.

## V. DSB

source url: https://github.com/microsoft/dsb.git

DSB is a benchmark based on TPC-DS, it has complex data distribution and challenging yet semantically meaningful query templates (3 new templates). Thus we could generate data and queries following the TPC-DS tutorial.



