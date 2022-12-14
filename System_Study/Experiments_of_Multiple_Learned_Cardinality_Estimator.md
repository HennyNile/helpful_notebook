# Experiments of Multiple Learned Cardinality Estimator

## I. MSCN

### 1. Source Code

https://github.com/andreaskipf/learnedcardinalities.git

### 2. Requirements

Pytorch 1.0

Python 3.7

Numpy

mkl 2018

### 3. Running Screen Shot

![image-20221126165141592](..\pictures\MSCN_sythetic.png)

![image-20221126165156941](..\pictures\MSCN_scale.png)

![image-20221126165812381](..\pictures\MSCN_job_light.png)

## II. E2E-TLSTM

### 1. Source Code

https://github.com/postechdblab/learned-cardinality-estimation.git

### 2. Requirements

See https://github.com/postechdblab/learned-cardinality-estimation

### 3. Running Screen Shot

![image-20221129143833184](..\pictures\E2E_job_light.png)

## III. DeepDB

main reference: https://github.com/DataManagementLab/deepdb-public

### 1. Source Code

https://github.com/DataManagementLab/deepdb-public.git

### 2. Requirements

libpq-dev, gcc, python3-dev

```bahs
sudo apt install -y libpq-dev gcc python3-dev
```

postgres

```bash
sudo apt-get install postgresql
find / -name "pg_hba.conf*"
vim .../pg_hba.conf
	# initial 
	# TYPE  DATABASE        USER            ADDRESS                 METHOD
	# "local" is for Unix domain socket connections only
	local   all             postgres                                peer

	# modified
	# TYPE  DATABASE        USER            ADDRESS                 METHOD
	# "local" is for Unix domain socket connections only
	local   all             postgres                                trust
```

### 3. Setup

```
conda create <env_name> python==3.7
pip3 install -r requirements_python3.7.txt
```

Then follow https://github.com/DataManagementLab/deepdb-public.

### 4. Running Screen Shot

![image-20221202184801749](..\pictures\DeepDB_job_light_1.png)

![image-20221202190043331](..\pictures\DeepDB_job_light_2.png)

![image-20221202190121353](..\pictures\DeepDB_job_light_3.png)

![image-20221202190158219](..\pictures\DeepDB_job_light_4.png)

 ## IV. NeuroCard

### 1. Source Code

https://github.com/neurocard/neurocard

### 2. Requirements

### 3. Running Screen Shot

![image-20221202183342126](..\pictures\Neurocard_job_light_1.png)

![image-20221202190341685](..\pictures\Neurocard_job_light_2.png)

![image-20221202190510744](..\pictures\Neurocard_job_light_3.png)

![image-20221202190711067](..\pictures\Neurocard_job_light_4.png)

## V. FLAT

### 1. Source Code

https://github.com/wuziniu/FSPN

### 2. Requirements

1. The specified version of many packages in environment.yml doesn't exist now, then you need to remove the version info of such package to install them. 
2. Not all packages are needed, you could just run the code to see what packages are needed and just install them.

### 3. Running Screen Shot

![image-20221214202359120](..\pictures\flat_1.png)

## VI. Flow-loss

### 1. Source Code

https://github.com/learnedsystems/CEB.git

### 2. Requirements

#### a. modify postgres connection info:

```
tests/test_installation.py
errors = ppc.eval(qreps, preds, samples_type="test",
            result_dir=None, user = "ceb", db_name = "imdb",
            db_host = "localhost", port = 5432,
            alg_name = "test")

main.py
    parser.add_argument("--user", type=str, required=False,
            default="ceb")
```

#### b. Setup postgresql-server-dev-all

``` 
sudo apt-get installpostgresql-server-dev-all
```

#### c. Setup pg_hint_plan

```bash
# check version of pg
postgres =# show server_version;

# build pg_hint_plan
git clone https://github.com/parimarjan/pg_hint_plan.git
cd pg_hint_plan
git checkout #appriorate branch
make
make install
```

### 3. Running Screen Shot

 ```bash
 python3 main.py --query_templates 1a,2a --algs postgres --eval_fns qerr,ppc,plancost --query_dir queries/imdb
 ```

![image-20221209215514942](..\pictures\flow_loss_1.png)

```bash
python3 main.py --query_templates all_jobm  --algs postgres --eval_fns qerr,ppc,plancost --query_dir queries/jobm
```

![image-20221209220659761](..\pictures\flow_loss_2.png)

```bash
python3 main.py --query_templates all_jobm  --algs true  --eval_fns qerr,ppc,plancost --query_dir queries/jobm
```

![image-20221209220822991](..\pictures\flow_loss_3.png)