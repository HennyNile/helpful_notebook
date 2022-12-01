# Setup Dataset used in Benchmarks

## I. JOB

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

## II. Official IMDB Dataset

**Main Reference**: [Loading IMDB data into postgresql](https://dbastreet.com/?p=1426)

### 1. Requirements

#### a. pip

```
sudo apt install python3-pip
```

#### b. libpq-dev

```
sudo apt-get install libpq-dev
```

#### c. Psycopg2

```
pip3 install Psycopg2
```

#### d. imdbpy

```
pip3 install imdbpy
```

#### e. s32cinemagoer.py

This is a script  that imports the s3 dataset distributed by IMDb into a SQL database.

https://github.com/cinemagoer/cinemagoer/blob/master/bin/s32cinemagoer.py

### 2. Get the Source

https://datasets.imdbws.com/

```
wget https://datasets.imdbws.com/name.basics.tsv.gz
wget https://datasets.imdbws.com/title.akas.tsv.gz
wget https://datasets.imdbws.com/title.basics.tsv.gz
wget https://datasets.imdbws.com/title.crew.tsv.gz
wget https://datasets.imdbws.com/title.episode.tsv.gz
wget https://datasets.imdbws.com/title.principals.tsv.gz
wget https://datasets.imdbws.com/title.ratings.tsv.gz
```

### 3. Load Data

#### a. Create Database

```
postgres# create database imdb;
```

#### b. Run Script

```bash
# template
python3 s32cinemagoer.py <Path of dir of IMDB Databaset> postgresql://username:password@dbhostname/dbname

# practical
python3 s32cinemagoer.py ~/workspace/0.LBO/IMDB/data postgresql://postgres_15_sc:postgres@localhost:5431/imdb
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



