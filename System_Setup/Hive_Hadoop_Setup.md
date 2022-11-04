# Setup Hadoop(Pseudo-Distributed) and Hive on Ubuntu

## Requirement 

Java

SSH

Hadoop

PostgreSQL (as the metastore of Hive) 

Hive

## Download and decompress compressed packages

Hadoop: https://www.apache.org/dyn/closer.cgi/hadoop/common/

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

0. 把JAVA_HOME，HADOOP_HOME和相关目录加进环境变量（~/.profile）

   ```bash
   export JAVA_HOME=<JAVA_HOME>
   export PATH=$JAVA_HOME/bin:$PATH
   export HADOOP_HOME=<HADOOP_HOME>
   export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
   ```

1. 编辑```$HADOOP_HOME/etc/hadoop/hadoop-env.sh```

   ```bash
   # set to the root of your Java installation
   export JAVA_HOME='<JAVA_HOME>'
   ```

2. 编辑```$HADOOP_HOME/etc/hadoop/core-site.xml```

   ```xml
   <configuration>
       <property>
           <name>fs.defaultFS</name>
           <value>hdfs://localhost:9000</value>
       </property>
   </configuration>
   ```

3. 编辑```$HADOOP_HOME/etc/hadoop/hdfs-site.xml```

   ```xml
   <configuration>
       <property>
           <name>dfs.replication</name>
           <value>1</value>
       </property>
   </configuration>
   ```

4. 格式化file system

   ```bash
   $ bin/hdfs namenode -format
   ```

5. 开启Namenode和DataNode的守护进程，可以通过网页端浏览Namenode，默认网址为http://localhost:9870/。

   ```bash
   $ sbin/start-dfs.sh
   ```

6. 编辑```$HADOOP_HOME/etc/hadoop/mapred-site.xml```

   ```xml
   <configuration>
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
       <property>
           <name>mapreduce.application.classpath</name>
           <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
       </property>
   </configuration>
   ```

7. 编辑```$HADOOP_HOME/etc/hadoop/yarn-site.xml```

   ```xml
   <configuration>
       <property>
           <name>yarn.nodemanager.aux-services</name>
           <value>mapreduce_shuffle</value>
       </property>
       <property>
           <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
       </property>
   </configuration>
   ```

8. 开启ResourceManager和NodeManager的守护进程，可以通过网页端浏览ResourceManager，默认网址为http://localhost:8088/。

   ```bash
    $ sbin/start-yarn.sh
   ```

9. 使用结束后，关闭所有守护进程

   ```bash
    $ sbin/stop-all.sh
   ```

10. 同9，可通过以下命令开启Hadoop和yarn所有进程

    ```bash
     $ sbin/start-all.sh

### III. Setup PostgreSQL for Hive MetaStore

Reference: https://dev.mysql.com/doc/refman/8.0/en/linux-installation-debian.html

1. 安装PostgreSQL

   ```bash
   $ sudo apt-get update
   $ sudo apt-get -y install postgres 
   ```

2. 给默认用户postgres设置密码, 密码设置为’postgres‘。

   ```bash
   $ sudo -u postgres psql # go to postgresql console
   postgres=# ALTER USERT postgres PASSWORD 'myPassword'
   ```

3. 为hive metastore创建数据库，数据库密码为‘hivemetastoredb’。

   ```bash
   $ sudo -i -u postgres
   $ createdb -h localhost -p 5432 -U postgres --password hivemetastoredb
   ```

### IV. Setup Hive

Reference: http://www.sqlnosql.com/install-hive-on-hadoop-3-xx-on-ubuntu-with-postgresql-database/

0. 把HIVE_HOME和相关目录加进环境变量（~/.profile）

   ```bash
   export HIVE_HOME=~/env/hive/apache-hive-3.1.3-bin
   export PATH=$HIVE_HOME/bin:$PATH
   ```

1. 在HDFS上创建Hive目录，并修改权限

   ```bash
   # warehouse is the dir to store tables and data related to hive
   $ hdfs dfs -mkdir -p /user/hive/warehouse 
   $ hdfs dfs -mkdir /tmp
   
   # give write permission to new dirs
   $ hdfs dfs -chmod g+w /user/hive/warehouse
   $ hdfs dfs -chmod g+w /tmp
   ```

2. 编辑`$HIVE_HOME/conf/hive-env.sh`

   ```bash
   cp $HIVE_HOME/conf/hive-env.sh.template $HIVE_HOME/conf/hive-env.sh
   vim $HIVE_HOME/conf/hive-env.sh
   
   export HADOOP_HOME=~/env/hadoop/hadoop-3.3.4
   export HIVE_CONF_DIR=~/env/hive/apache-hive-3.1.3-bin/conf
   ```

3. 编辑`$HIVE_HOME/conf/hive-site.sh`

   ```bash
   cp $HIVE_HOME/hcatalog/etc/hcatalog/proto-hive-site.xml $HIVE_HOME/conf/hive-site.xml
   vim $HIVE_HOME/conf/hive-site.xml
   ```

   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
   <configuration>
   <property>
   <name>hive.metastore.local</name>
   <value>true</value>
   </property>
   
   <property>
   <name>hive.metastore.warehouse.dir</name>
   <value>/user/hive/warehouse</value>
   </property>
   
   <property>
   <name>javax.jdo.option.ConnectionDriverName</name>
   <value>org.postgresql.Driver</value>
   </property>
   
   <property>
   <name>javax.jdo.option.ConnectionURL</name>
   <value>jdbc:postgresql://localhost:5432/hivemetastoredb</value>
   </property>
   
   <property>
   <name>javax.jdo.option.ConnectionUserName</name>
   <value>postgres</value>
   </property>
   
   <property>
   <name>javax.jdo.option.ConnectionPassword</name>
   <value>postgres</value>
   </property>
   
   <property>
   <name>hive.server2.thrift.port</name>
   <value>10000</value>
   </property>
   
   <property>
   <name>hive.server2.enable.doAs</name>
   <value>true</value>
   </property>
   
   <property>
   <name>hive.execution.engine</name>
   <value>mr</value>
   </property>
   
   <property>
   <name>hive.metastore.port</name>
   <value>9083</value>
   </property>
   
   <property>
   <name>mapreduce.input.fileinputformat.input.dir.recursive</name>
   <value>true</value>
   </property>
   </configuration>
   ```

4. 下载[PostgreSQL JDBC Driver](https://jdbc.postgresql.org/download/)，并移动至`$HIVE_HOME/lib`。

5. 在pg中创建Hive Schema

   ```bash
   $ $HIVE_HOME/bin/schematool -initSchema -dbType postgres
   ```

6. 运行hive

   ```bash
    $ $HIVE_HOME/bin/hive
   ```

## Normal Use

1. 开启Hadoop和yarn

   ```bash
   $ $HADOOP_HOME/sbin/start-all.sh
   ```

2. 直接运行hive

   ```bash
   $ hive
   ```

3. 运行hiveserver2和hive metastore

   ```bash
   # start hiveserver2, default port 10000
   $ mkdir ~/hiveserver2log
   $ cd ~/hiveserver2log
   $ nohup $HIVE_HOME/bin/hive --service hiveserver2 &
   
   # after start Hiveserver2, you can browser Hiveserver3 Web UI in localhost:10002
   
   # start metastore, default port 9083
   $ mkdir ~/hivemetastorelog
   $ cd ~/hivemetastorelog
   $ nohup hive --service metastore &
   ```

4. 常用网页接口

​		HDFS Namenode Information:  localhost:9870/

​		Yarn: localhost:8088/

​		Hiveserver2: localhost:10002/





 















