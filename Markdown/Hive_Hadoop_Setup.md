# Setup Hadoop(Pseudo-Distributed) and Hive on Ubuntu

## Requirement 

Java

SSH

Hadoop

MySQL (as the metastore of Hive) 

Hive

## Download and decompress compressed packages

Hadoop: https://www.apache.org/dyn/closer.cgi/hadoop/common/

MySQL: https://dev.mysql.com/downloads/mysql/

Hive: https://www.apache.org/dyn/closer.cgi/hive/

## Setups

### I. Setup SSH

1. 安装ssh

   ```bash
   sudo apt-get install ssh
   ```

2. setup passphraseless ssh

   测试是否可以直接ssh连接本机；

   ```bash
   ssh localhost
   ```

   如果上述操作没有直接连接本机，则将本机公钥加入授权的主机

   ```bash
   ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
   cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   chmod 0600 ~/.ssh/authorized_keys
   ```

### II. Setup Hadoop

Reference: https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html

1. 

### III. Setup MySQL

### IV. Setup Hive



 















