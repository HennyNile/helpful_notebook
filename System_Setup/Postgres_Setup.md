# Setup Postgres on Linux

## I. Requirements

You could determine the name of a software in **ubuntu software center**.

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

```
./configure
make
su
make install
adduser postgres_15_sc
mkdir -p /usr/local/pgsql/data
chown postgres_15_sc /usr/local/pgsql/data
su - postgres_15_sc
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
/usr/local/pgsql/bin/createdb test
/usr/local/pgsql/bin/psql test
```

### 2. Complete Version

https://www.postgresql.org/docs/current/install-procedure.html

### 3. Modify Port

```bash
# find postgres config files
sudo find / -name 'postgresql.conf'

# find port and modify it to 5431
```

### 4. Start and Stop Service

```bash
# start service
su - postgres_15_sc
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start

# stop service
su - postgres_15_sc
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile stop
```

## IV. Post-installation Check

### 1. Select databases

```
# \l 
```

### 2. Connect to Postgres

```
/usr/local/pgsql/bin/psql -h localhost -p 5431 -U postgres_15_sc postgres
/usr/local/pgsql/bin/psql -h localhost -p 5431 -U postgres_15_sc imdb
/usr/local/pgsql/bin/psql -h localhost -p 5431 -U postgres_15_sc ssb_1
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







