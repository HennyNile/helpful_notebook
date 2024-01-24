# Setup Postgres on Linux

## I. Requirements

You could determine the name of a software in **ubuntu software center**. 

If you first make then install these missing packages, configure again before next m

### 1. GNU make

```
sudo apt-get install make
```

### 2. Flex

```
sudo apt-get install flex
```

### 3. Bison

```
sudo apt-get install bison
```

### 4. readline

```
sudo apt install libreadline-dev
```

### 5. zlib

```
sudo apt install zlib1g

sudo apt install zlib1g-dev
```

## II. Getting the Source

Get source from https://www.postgresql.org/ftp/source/.

```
tar xf postgresql-version.tar.gz
```

## III. Installation Procedure

### 1. Short Version

```bash
./configure --bindir=[absolute path of bin dir] --datadir=[absolute path of data dir 1] --libdir=[absolute path of lib dir] --enable-debug
# example: ./configure --bindir=/home/dbgroup/workspace/liqilong/LBO/pg15/bin --datadir=/home/dbgroup/workspace/liqilong/LBO/pgdata --libdir=/home/dbgroup/workspace/liqilong/LBO/pg15/lib --enable-debug

make
make install
[bin dir]/initdb -D [data dir 2]
[bin dir]/pg_ctl -D [data dir 2] -l logfile start
[bin dir]/createdb test
[bin dir]/psql test
```

### 2. Complete Version

https://www.postgresql.org/docs/current/install-procedure.html

### 3. Modify Port

```bash
# find postgres config files
sudo find / -name 'postgresql.conf'

# find port and modify it to 5431
```

### 4. (Optional) FATAL: Peer authentication failed for user "postgres" 

https://blog.csdn.net/liyazhen2011/article/details/88977424

### 5. Start and Stop Service

```bash
# start service
#su - postgres_15_sc
#/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
bin/pg_ctl -D [datadir] -l logfile start

# stop service
#su - postgres_15_sc
#/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile stop
bin/pg_ctl -D [datadir] -l logfile stop
```

## IV. Post-installation Check

### 1. Select databases

```
# \l 
```

### 2. Connect to Postgres

```
bin/psql -h [host] -p [port] -U [username] [database]
bin/psql -h localhost -p 5431 -U postgres_15_sc imdb
```

### 3. Select Database

```
\c <database_name>
```

### 4. Table list

```
# show all tables in current database
\d

# show specific information of a table
\d <table_name>
```

### 5. Analyze Tables

```
ANALYZE [ VERBOSE ] [ table_and_columns [, ...] ]
```

### 6. Modify config parameters

```postgresql
--- show parameters values
postgres=# select current_setting('work_mem');

--- set paramters and reload config parameters
--- set_config ( setting_name text, new_value text, is_local boolean ) → text
--- Sets the parameter setting_name to new_value, and returns that value. If is_local is true, the new value will only apply during the current transaction. If you want the new value to apply for the rest of the current session, use false instead. 
postgres=# select set_config('work_mem', '100MB', false);
postgres=# alter system set work_mem='100MB';
postgres=# select pg_reload_conf();
```

 





