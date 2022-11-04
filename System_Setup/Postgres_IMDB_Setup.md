# Load IMDB Dataset into Postgres

**Main Reference**: [Loading IMDB data into postgresql](https://dbastreet.com/?p=1426)

# I. Requirements

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

