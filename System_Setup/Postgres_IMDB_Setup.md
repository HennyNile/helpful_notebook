# IMDB Dataset used in Benchmark JOB

Dataset Resource: https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT

Main Reference: https://www.postgresql.org/docs/7.1/app-pgdump.html, https://www.postgresql.org/docs/current/app-pgrestore.html

## I. Getting the Source

Download from https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT.

## II. Load the Data 

The data is dumped from postgres database with command 

```bash
pg_dump -Fc
```

Then we use pg_restore to restore data

```bash
su - postgres_15_sc
pg_restore -h localhost -p 5431 -U postgres_15_sc -C -d postgres /tmp/imdb_pg11
```

# Official IMDB Dataset

**Main Reference**: [Loading IMDB data into postgresql](https://dbastreet.com/?p=1426)

## I. Requirements

### 1. pip

```
sudo apt install python3-pip
```

### 2. libpq-dev

```
sudo apt-get install libpq-dev
```

### 3. Psycopg2

```
pip3 install Psycopg2
```

### 4. imdbpy

```
pip3 install imdbpy
```

### 5. s32cinemagoer.py

This is a script  that imports the s3 dataset distributed by IMDb into a SQL database.

https://github.com/cinemagoer/cinemagoer/blob/master/bin/s32cinemagoer.py

## II. Getting the Source

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

## III. Load Data

### 1. Create Database

```
postgres# create database imdb;
```

### 2. Run Script

```bash
# template
python3 s32cinemagoer.py <Path of dir of IMDB Databaset> postgresql://username:password@dbhostname/dbname

# practical
python3 s32cinemagoer.py ~/workspace/0.LBO/IMDB/data postgresql://postgres_15_sc:postgres@localhost:5431/imdb
```
