### 4. 单机部署

![image-20210627144457088](./imgs/image-20210627144457088.png)

#### 1. 本地部署



##### 1. 依赖

![image-20210614235313834](./imgs/image-20210614235313834.png)

[Prerequisites](https://airflow.apache.org/docs/apache-airflow/stable/installation.html)



##### 2. 升级Python3.7

> 以 root 用户运行

```bash
#!/bin/bash
# File: upgrade_python37.sh
# User: root
# Os: CentOS 7.9

# 1. Install required package
yum install -y gcc gcc-c++ python-devel openssl-devel zlib-devel readline-devel libffi-devel wget

# 2. Install Python-3.7
#wget https://www.python.org/ftp/python/3.7.10/Python-3.7.10.tar.xz
wget https://zhengyansheng.oss-cn-beijing.aliyuncs.com/Python-3.7.10.tar.xz
tar xf Python-3.7.10.tar.xz
cd Python-3.7.10
#./configure --prefix=/usr/local/python37 --enable-optimizations --with-ssl --enable-loadable-sqlite-extensions
./configure --prefix=/usr/local/python37 --enable-optimizations --with-ssl
make -j 4
make install

# 3. Link python3.7 to python
unlink /usr/bin/python
ln -sv /usr/local/python37/bin/python3.7 /usr/bin/python

# 4. Add pip.conf file
#cat > /etc/pip.conf << EOF
#[global]
#trusted-host = pypi.douban.com
#index-url = http://pypi.douban.com/simple

#[list]
#format=columns
#EOF

cat > /etc/pip.conf << EOF
[global]
trusted-host = mirrors.aliyun.com
index-url = http://mirrors.aliyun.com/pypi/simple/

[list]
format=columns
EOF

# 5. Add local env
echo export PATH="/usr/local/python37/bin:$PATH" >> ~/.bashrc
source ~/.bashrc
pip3.7 install --upgrade pip==20.2.4 # fix https://github.com/apache/airflow/issues/12838

# 6. View version
python --version
pip3.7 --version 
```



注意事项:

*由于将OS系统默认的Python版本更改了，导致系统自带的命令行工具(yum/ urlgrabber-ext-down/ yum-config-manager)无法直接使用，需要做更改才行*

```python
"""
1. vi /usr/bin/yum
2. vi /usr/libexec/urlgrabber-ext-down
3. vi /usr/bin/yum-config-manager
"""
```



##### 3. 部署MySQL 5.7数据库

###### 1. 安装

```bash
#!/bin/bash
# File: install_mysql57.sh
# User: root
# Os: CentOS 7.9
# Reference: https://tecadmin.net/install-mysql-5-7-centos-rhel/

# 1. Install yum source
yum localinstall -y https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm

# 2. Install mysql
#yum install -y mysql-community-server
#yum localinstall *.rpm # yum install --downloadonly --downloaddir=./ mysql-community-server
wget https://zhengyansheng.oss-cn-beijing.aliyuncs.com/mysql-yum-57.tar.gz
tar xf mysql-yum-57.tar.gz
cd mysql-yum-57
yum localinstall -y *.rpm
```

###### 2. 启动数据库

```bash
# 1. Start mysql
systemctl start mysqld.service

# 2. view mysql login password
grep 'A temporary password' /var/log/mysqld.log |tail -1

# 3. set secure option
/usr/bin/mysql_secure_installation

# 4. view version
mysql -V

"""
echo explicit_defaults_for_timestamp=1 >> /etc/my.cnf
systemctl restart mysqld.service
"""
```

###### 3. 创建数据库

```bash
> mysql -uroot -p<xxx>
set global validate_password_policy=LOW;
set global validate_password_length=6;
alter user user() identified by "123456";
CREATE DATABASE `airflow` /*!40100 DEFAULT CHARACTER SET utf8 */;
CREATE USER 'airflow_user'@'localhost' IDENTIFIED BY 'airflow12345678';
GRANT ALL ON airflow.* TO 'airflow_user'@'localhost';
FLUSH PRIVILEGES;
```

##### 4. 部署Redis 6.x数据库

###### 1. 安装

```bash
# 1. Install remi yum repo
yum install -y epel-release yum-utils
yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum-config-manager --enable remi

# 2. Install redis latest version
yum install -y redis
```

###### 2. 配置

```bash
# vi /etc/redis.conf
bind 0.0.0.0
```

###### 3. 启动

```bash
# 1. Start redis
systemctl start redis && systemctl enable redis
systemctl status redis

# 2. View redis
ps -ef |grep redis

# 3. Test
redis-cli ping

# 4. View version
redis-cli --version
```

##### 5. 部署airflow

###### 1. 安装

```bash
# 1. Set env
export AIRFLOW_HOME=~/airflow

# 2. Install apache-airflow 2.1.0
AIRFLOW_VERSION=2.1.0
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
# For example: https://raw.githubusercontent.com/apache/airflow/constraints-2.1.0/constraints-3.6.txt # 有可能网络不通，见<<3. 注意事项>>
pip3.7 install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

###### 2. 初始化数据库

```bash
# 1. Set up database
## https://airflow.apache.org/docs/apache-airflow/2.1.0/howto/set-up-database.html#
pip3.7 install pymysql
airflow config get-value core sql_alchemy_conn  # Error, but create ~/airflow directory

# 2. Initialize the database
"""
# vi ~/airflow/airflow.cfg
[core]
sql_alchemy_conn = mysql+pymysql://airflow_user:airflow12345678@localhost:3306/airflow
"""

airflow db init

"""
...

Initialization done
"""

```

###### 3. 创建用户

```bash
# Create superuser
airflow users create \
    --username admin \
    --firstname zheng \
    --lastname yansheng \
    --role Admin \
    --email zhengyansheng@gmail.com
```

###### 4. 启动服务

```bash
# start the web server, default port is 8080 -D
airflow webserver --port 8080 

# start the scheduler
# open a new terminal or else run webserver with ``-D`` option to run it as a daemon
airflow scheduler

# visit localhost:8080 in the browser and use the admin account you just
# created to login. Enable the example_bash_operator dag in the home page
```

###### 5. 管理后台

**Web admin**

<img src="./imgs/image-20210620131951045.png" alt="image-20210620131951045" style="zoom:50%;" />



**Dashboard**

<img src="./imgs/image-20210620132027077.png" alt="image-20210620132027077" style="zoom:50%;" />

##### 6. 分布式部署

###### 1. 安装

```bash
pip install 'apache-airflow[celery]'
pip install celery[redis]
```



###### 2. 设置executor

```bash
[core]
# The executor class that airflow should use. Choices include
# ``SequentialExecutor``, ``LocalExecutor``, ``CeleryExecutor``, ``DaskExecutor``,
# ``KubernetesExecutor``, ``CeleryKubernetesExecutor`` or the
# full import path to the class when using a custom executor.
# executor = SequentialExecutor
executor = CeleryExecutor


[celery]
# broker_url = redis://redis:6379/0
broker_url = redis://localhost:6379/0

# result_backend = db+postgresql://postgres:airflow@postgres/airflow
result_backend = redis://localhost:6379/0
```



###### 3. 启动

```bash
# 1. Start webserver
airflow webserver -p 8000

# 2. Start scheduler
airflow scheduler

# 3. Start celery worker
airflow celery worker

# 4. Start celery flower
airflow celery flower
```



###### 4. 管理页面

**Webserver**

![image-20210620145955347](./imgs/image-20210620145955347.png)



**flower**

![image-20210620145723298](./imgs/image-20210620145723298.png)

##### 7. 演示

**启动Dag**

![image-20210626140139881](./imgs/image-20210626140139881.png)



**查看scheduler日志**

![image-20210626140236587](./imgs/image-20210626140236587.png)

**查看Worker日志**

![image-20210626140305869](./imgs/image-20210626140305869.png)

#### 2. Docker部署



##### 1. 安装Docker

> https://docs.docker.com/engine/install/centos/

```bash
# 1. Clear old package
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# 2. Install dependent package
yum install -y yum-utils

# 3. Install docker-ce
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
```

##### 2. 配置Docker

```bash
# 1. Setup docker configuration
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
EOF
```



##### 3. 启动Docker

```bash
# 1. Start docker
systemctl start docker && systemctl enable docker

# 2. Run a hello-world container
docker run hello-world

# 3. View docker version
docker version
```

##### 4.  安装docker-compose

> https://docs.docker.com/compose/install/

```bash
# 1. Download docker-compose binary file
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 2. Add execute permission
chmod +x /usr/local/bin/docker-compose

# 3. link docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# 4. View version
docker-compose --version
```

##### 5. 部署airflow

```bash
# 1. Create docker-compose directory
mkdir deploy
cd deploy 

# 2. Download docker-compose.yaml
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.0.1/docker-compose.yaml'

# 3. Init airflow, database migrations and create the first user account
# dags 存放自定义的dag文件
# logs  存放服务的日志文件
# plugins 存放插件文件
mkdir ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0\nAIRFLOW_IMAGE_NAME=apache/airflow:2.1.0 " > .env
# export AIRFLOW_IMAGE_NAME=apache/airflow:2.1.0  # fix https://github.com/apache/airflow/issues/15312
docker-compose up airflow-init

# 4. Start all services
docker-compose up
docker-compose stop
#docker-compose up -d #后台运行

# 5. View all containers
docker ps
```

###### 1. init log

```bash
# docker-compose up airflow-init
Pulling airflow-init (apache/airflow:2.1.0)...
2.1.0: Pulling from apache/airflow
69692152171a: Already exists
66a3c154490a: Already exists
a1aac2599825: Pull complete
b7d9b1f22943: Pull complete
77cacf443dcc: Pull complete
72f126efb85d: Pull complete
c6eb6ecb21b3: Pull complete
a5cb083e5376: Pull complete
a08a4b96aa37: Pull complete
5a9395e183b7: Pull complete
54e343cceb4f: Pull complete
05bbdd5c2d0f: Pull complete
d3fa3061e610: Pull complete
d607824ebc97: Pull complete
Digest: sha256:a2139c0624138a965f5664a9595c02a98c65f4018e564f4a2b56e3d300d429ac
Status: Downloaded newer image for apache/airflow:2.1.0
d_postgres_1 is up-to-date
d_redis_1 is up-to-date
Recreating d_airflow-init_1 ... done
Attaching to d_airflow-init_1
airflow-init_1       | The container is run as root user. For security, consider using a regular user account.
airflow-init_1       | BACKEND=postgresql+psycopg2
airflow-init_1       | DB_HOST=postgres
airflow-init_1       | DB_PORT=5432
airflow-init_1       |
airflow-init_1       | [2021-06-20 08:24:02,045] {cli_action_loggers.py:105} WARNING - Failed to log action with (psycopg2.errors.UndefinedTable) relation "log" does not exist
airflow-init_1       | LINE 1: INSERT INTO log (dttm, dag_id, task_id, event, execution_dat...
airflow-init_1       |                     ^
airflow-init_1       |
airflow-init_1       | [SQL: INSERT INTO log (dttm, dag_id, task_id, event, execution_date, owner, extra) VALUES (%(dttm)s, %(dag_id)s, %(task_id)s, %(event)s, %(execution_date)s, %(owner)s, %(extra)s) RETURNING log.id]
airflow-init_1       | [parameters: {'dttm': datetime.datetime(2021, 6, 20, 8, 24, 1, 820015, tzinfo=Timezone('UTC')), 'dag_id': None, 'task_id': None, 'event': 'cli_upgradedb', 'execution_date': None, 'owner': 'root', 'extra': '{"host_name": "31cf005a3dfc", "full_command": "[\'/home/airflow/.local/bin/airflow\', \'db\', \'upgrade\']"}'}]
airflow-init_1       | (Background on this error at: http://sqlalche.me/e/13/f405)
airflow-init_1       | DB: postgresql+psycopg2://airflow:***@postgres/airflow
airflow-init_1       | [2021-06-20 08:24:02,366] {db.py:695} INFO - Creating tables
airflow-init_1       | INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
airflow-init_1       | INFO  [alembic.runtime.migration] Will assume transactional DDL.
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade  -> e3a246e0dc1, current schema
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade e3a246e0dc1 -> 1507a7289a2f, create is_encrypted
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 1507a7289a2f -> 13eb55f81627, maintain history for compatibility with earlier migrations
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 13eb55f81627 -> 338e90f54d61, More logging into task_instance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 338e90f54d61 -> 52d714495f0, job_id indices
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 52d714495f0 -> 502898887f84, Adding extra to Log
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 502898887f84 -> 1b38cef5b76e, add dagrun
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 1b38cef5b76e -> 2e541a1dcfed, task_duration
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 2e541a1dcfed -> 40e67319e3a9, dagrun_config
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 40e67319e3a9 -> 561833c1c74b, add password column to user
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 561833c1c74b -> 4446e08588, dagrun start end
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 4446e08588 -> bbc73705a13e, Add notification_sent column to sla_miss
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade bbc73705a13e -> bba5a7cfc896, Add a column to track the encryption state of the 'Extra' field in connection
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade bba5a7cfc896 -> 1968acfc09e3, add is_encrypted column to variable table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 1968acfc09e3 -> 2e82aab8ef20, rename user table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 2e82aab8ef20 -> 211e584da130, add TI state index
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 211e584da130 -> 64de9cddf6c9, add task fails journal table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 64de9cddf6c9 -> f2ca10b85618, add dag_stats table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade f2ca10b85618 -> 4addfa1236f1, Add fractional seconds to mysql tables
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 4addfa1236f1 -> 8504051e801b, xcom dag task indices
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 8504051e801b -> 5e7d17757c7a, add pid field to TaskInstance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 5e7d17757c7a -> 127d2bf2dfa7, Add dag_id/state index on dag_run table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 127d2bf2dfa7 -> cc1e65623dc7, add max tries column to task instance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade cc1e65623dc7 -> bdaa763e6c56, Make xcom value column a large binary
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade bdaa763e6c56 -> 947454bf1dff, add ti job_id index
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 947454bf1dff -> d2ae31099d61, Increase text size for MySQL (not relevant for other DBs' text types)
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade d2ae31099d61 -> 0e2a74e0fc9f, Add time zone awareness
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade d2ae31099d61 -> 33ae817a1ff4, kubernetes_resource_checkpointing
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 33ae817a1ff4 -> 27c6a30d7c24, kubernetes_resource_checkpointing
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 27c6a30d7c24 -> 86770d1215c0, add kubernetes scheduler uniqueness
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 86770d1215c0, 0e2a74e0fc9f -> 05f30312d566, merge heads
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 05f30312d566 -> f23433877c24, fix mysql not null constraint
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade f23433877c24 -> 856955da8476, fix sqlite foreign key
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 856955da8476 -> 9635ae0956e7, index-faskfail
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 9635ae0956e7 -> dd25f486b8ea, add idx_log_dag
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade dd25f486b8ea -> bf00311e1990, add index to taskinstance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 9635ae0956e7 -> 0a2a5b66e19d, add task_reschedule table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 0a2a5b66e19d, bf00311e1990 -> 03bc53e68815, merge_heads_2
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 03bc53e68815 -> 41f5f12752f8, add superuser field
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 41f5f12752f8 -> c8ffec048a3b, add fields to dag
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade c8ffec048a3b -> dd4ecb8fbee3, Add schedule interval to dag
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade dd4ecb8fbee3 -> 939bb1e647c8, task reschedule fk on cascade delete
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 939bb1e647c8 -> 6e96a59344a4, Make TaskInstance.pool not nullable
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 6e96a59344a4 -> d38e04c12aa2, add serialized_dag table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade d38e04c12aa2 -> b3b105409875, add root_dag_id to DAG
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 6e96a59344a4 -> 74effc47d867, change datetime to datetime2(6) on MSSQL tables
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 939bb1e647c8 -> 004c1210f153, increase queue name size limit
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade c8ffec048a3b -> a56c9515abdc, Remove dag_stat table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade a56c9515abdc, 004c1210f153, 74effc47d867, b3b105409875 -> 08364691d074, Merge the four heads back together
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 08364691d074 -> fe461863935f, increase_length_for_connection_password
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade fe461863935f -> 7939bcff74ba, Add DagTags table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 7939bcff74ba -> a4c2fd67d16b, add pool_slots field to task_instance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade a4c2fd67d16b -> 852ae6c715af, Add RenderedTaskInstanceFields table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 852ae6c715af -> 952da73b5eff, add dag_code table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 952da73b5eff -> a66efa278eea, Add Precision to execution_date in RenderedTaskInstanceFields table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade a66efa278eea -> da3f683c3a5a, Add dag_hash Column to serialized_dag table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade da3f683c3a5a -> 92c57b58940d, Create FAB Tables
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 92c57b58940d -> 03afc6b6f902, Increase length of FAB ab_view_menu.name column
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 03afc6b6f902 -> cf5dc11e79ad, drop_user_and_chart
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade cf5dc11e79ad -> bbf4a7ad0465, Remove id column from xcom
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade bbf4a7ad0465 -> b25a55525161, Increase length of pool name
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade b25a55525161 -> 3c20cacc0044, Add DagRun run_type
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 3c20cacc0044 -> 8f966b9c467a, Set conn_type as non-nullable
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 8f966b9c467a -> 8d48763f6d53, add unique constraint to conn_id
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 8d48763f6d53 -> e38be357a868, Add sensor_instance table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade e38be357a868 -> b247b1e3d1ed, Add queued by Job ID to TI
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade b247b1e3d1ed -> e1a11ece99cc, Add external executor ID to TI
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade e1a11ece99cc -> bef4f3d11e8b, Drop KubeResourceVersion and KubeWorkerId
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade bef4f3d11e8b -> 98271e7606e2, Add scheduling_decision to DagRun and DAG
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 98271e7606e2 -> 52d53670a240, fix_mssql_exec_date_rendered_task_instance_fields_for_MSSQL
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 52d53670a240 -> 364159666cbd, Add creating_job_id to DagRun table
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 364159666cbd -> 45ba3f1493b9, add-k8s-yaml-to-rendered-templates
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 45ba3f1493b9 -> 849da589634d, Prefix DAG permissions.
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 849da589634d -> 2c6edca13270, Resource based permissions.
airflow-init_1       | [2021-06-20 08:24:09,121] {manager.py:784} WARNING - No user yet created, use flask fab command to do it.
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 2c6edca13270 -> 61ec73d9401f, Add description field to connection
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 61ec73d9401f -> 64a7d6477aae, fix description field in connection to be text
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 64a7d6477aae -> e959f08ac86c, Change field in DagCode to MEDIUMTEXT for MySql
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade e959f08ac86c -> 82b7c48c147f, Remove can_read permission on config resource for User and Viewer role
airflow-init_1       | [2021-06-20 08:24:12,534] {manager.py:784} WARNING - No user yet created, use flask fab command to do it.
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 82b7c48c147f -> 449b4072c2da, Increase size of connection.extra field to handle multiple RSA keys
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 449b4072c2da -> 8646922c8a04, Change default pool_slots to 1
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 8646922c8a04 -> 2e42bb497a22, rename last_scheduler_run column
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 2e42bb497a22 -> 90d1635d7b86, Increase pool name size in TaskInstance
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade 90d1635d7b86 -> e165e7455d70, add description field to variable
airflow-init_1       | INFO  [alembic.runtime.migration] Running upgrade e165e7455d70 -> a13f7613ad25, Resource based permissions for default FAB views.
airflow-init_1       | [2021-06-20 08:24:14,428] {manager.py:784} WARNING - No user yet created, use flask fab command to do it.
airflow-init_1       | Upgrades done
airflow-init_1       | [2021-06-20 08:24:21,196] {manager.py:784} WARNING - No user yet created, use flask fab command to do it.
airflow-init_1       | Admin user airflow created
airflow-init_1       | 2.1.0
d_airflow-init_1 exited with code 0
```

###### 2. up log

```bash
# docker-compose up
d_postgres_1 is up-to-date
d_redis_1 is up-to-date
Creating d_flower_1            ... done
Creating d_airflow-worker_1    ... done
Creating d_airflow-webserver_1 ... done
Starting d_airflow-init_1      ... done
Creating d_airflow-scheduler_1 ... done
Attaching to d_postgres_1, d_redis_1, d_airflow-init_1, d_airflow-webserver_1, d_airflow-worker_1, d_airflow-scheduler_1, d_flower_1
airflow-init_1       | The container is run as root user. For security, consider using a regular user account.
airflow-webserver_1  | The container is run as root user. For security, consider using a regular user account.
airflow-worker_1     | The container is run as root user. For security, consider using a regular user account.
airflow-scheduler_1  | The container is run as root user. For security, consider using a regular user account.
postgres_1           | The files belonging to this database system will be owned by user "postgres".
postgres_1           | This user must also own the server process.
postgres_1           |
postgres_1           | The database cluster will be initialized with locale "en_US.utf8".
postgres_1           | The default database encoding has accordingly been set to "UTF8".
postgres_1           | The default text search configuration will be set to "english".
postgres_1           |
postgres_1           | Data page checksums are disabled.
postgres_1           |
postgres_1           | fixing permissions on existing directory /var/lib/postgresql/data ... ok
postgres_1           | creating subdirectories ... ok
postgres_1           | selecting dynamic shared memory implementation ... posix
postgres_1           | selecting default max_connections ... 100
postgres_1           | selecting default shared_buffers ... 128MB
postgres_1           | selecting default time zone ... Etc/UTC
postgres_1           | creating configuration files ... ok
postgres_1           | running bootstrap script ... ok
postgres_1           | performing post-bootstrap initialization ... ok
postgres_1           | syncing data to disk ... ok
postgres_1           |
postgres_1           |
postgres_1           | Success. You can now start the database server using:
postgres_1           |
postgres_1           |     pg_ctl -D /var/lib/postgresql/data -l logfile start
postgres_1           |
postgres_1           | initdb: warning: enabling "trust" authentication for local connections
postgres_1           | You can change this by editing pg_hba.conf or using the option -A, or
postgres_1           | --auth-local and --auth-host, the next time you run initdb.
postgres_1           | waiting for server to start....2021-06-20 06:00:22.162 UTC [46] LOG:  starting PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
postgres_1           | 2021-06-20 06:00:22.163 UTC [46] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
postgres_1           | 2021-06-20 06:00:22.167 UTC [47] LOG:  database system was shut down at 2021-06-20 06:00:21 UTC
postgres_1           | 2021-06-20 06:00:22.171 UTC [46] LOG:  database system is ready to accept connections
postgres_1           |  done
postgres_1           | server started
postgres_1           | CREATE DATABASE
postgres_1           |
postgres_1           |
postgres_1           | /usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
postgres_1           |
postgres_1           | 2021-06-20 06:00:22.721 UTC [46] LOG:  received fast shutdown request
postgres_1           | waiting for server to shut down....2021-06-20 06:00:22.722 UTC [46] LOG:  aborting any active transactions
postgres_1           | 2021-06-20 06:00:22.724 UTC [46] LOG:  background worker "logical replication launcher" (PID 53) exited with exit code 1
postgres_1           | 2021-06-20 06:00:22.725 UTC [48] LOG:  shutting down
postgres_1           | 2021-06-20 06:00:22.738 UTC [46] LOG:  database system is shut down
postgres_1           |  done
postgres_1           | server stopped
postgres_1           |
postgres_1           | PostgreSQL init process complete; ready for start up.
postgres_1           |
postgres_1           | 2021-06-20 06:00:22.848 UTC [1] LOG:  starting PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
postgres_1           | 2021-06-20 06:00:22.848 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
postgres_1           | 2021-06-20 06:00:22.848 UTC [1] LOG:  listening on IPv6 address "::", port 5432
postgres_1           | 2021-06-20 06:00:22.850 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
postgres_1           | 2021-06-20 06:00:22.853 UTC [74] LOG:  database system was shut down at 2021-06-20 06:00:22 UTC
postgres_1           | 2021-06-20 06:00:22.857 UTC [1] LOG:  database system is ready to accept connections
postgres_1           |
postgres_1           | PostgreSQL Database directory appears to contain a database; Skipping initialization
postgres_1           |
postgres_1           | 2021-06-20 06:07:20.585 UTC [1] LOG:  starting PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
postgres_1           | 2021-06-20 06:07:20.585 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
postgres_1           | 2021-06-20 06:07:20.585 UTC [1] LOG:  listening on IPv6 address "::", port 5432
postgres_1           | 2021-06-20 06:07:20.587 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
postgres_1           | 2021-06-20 06:07:20.590 UTC [25] LOG:  database system was interrupted; last known up at 2021-06-20 06:05:23 UTC
postgres_1           | 2021-06-20 06:07:20.654 UTC [25] LOG:  database system was not properly shut down; automatic recovery in progress
postgres_1           | 2021-06-20 06:07:20.656 UTC [25] LOG:  redo starts at 0/15C7650
postgres_1           | 2021-06-20 06:07:20.656 UTC [25] LOG:  invalid record length at 0/15C7738: wanted 24, got 0
postgres_1           | 2021-06-20 06:07:20.656 UTC [25] LOG:  redo done at 0/15C7700
postgres_1           | 2021-06-20 06:07:20.664 UTC [1] LOG:  database system is ready to accept connections
postgres_1           | 2021-06-20 06:35:28.760 UTC [1] LOG:  received fast shutdown request
postgres_1           | 2021-06-20 06:35:28.764 UTC [1] LOG:  aborting any active transactions
postgres_1           | 2021-06-20 06:35:28.767 UTC [1] LOG:  background worker "logical replication launcher" (PID 31) exited with exit code 1
postgres_1           | 2021-06-20 06:35:28.768 UTC [26] LOG:  shutting down
postgres_1           | 2021-06-20 06:35:28.781 UTC [1] LOG:  database system is shut down
postgres_1           |
postgres_1           | PostgreSQL Database directory appears to contain a database; Skipping initialization
postgres_1           |
postgres_1           | 2021-06-20 08:10:03.658 UTC [1] LOG:  starting PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
postgres_1           | 2021-06-20 08:10:03.659 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
postgres_1           | 2021-06-20 08:10:03.659 UTC [1] LOG:  listening on IPv6 address "::", port 5432
postgres_1           | 2021-06-20 08:10:03.661 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
flower_1             | The container is run as root user. For security, consider using a regular user account.
postgres_1           | 2021-06-20 08:10:03.664 UTC [25] LOG:  database system was shut down at 2021-06-20 06:35:28 UTC
postgres_1           | 2021-06-20 08:10:03.671 UTC [1] LOG:  database system is ready to accept connections
postgres_1           | 2021-06-20 08:24:02.044 UTC [1228] ERROR:  relation "log" does not exist at character 13
postgres_1           | 2021-06-20 08:24:02.044 UTC [1228] STATEMENT:  INSERT INTO log (dttm, dag_id, task_id, event, execution_date, owner, extra) VALUES ('2021-06-20T08:24:01.820015+00:00'::timestamptz, NULL, NULL, 'cli_upgradedb', NULL, 'root', '{"host_name": "31cf005a3dfc", "full_command": "[''/home/airflow/.local/bin/airflow'', ''db'', ''upgrade'']"}') RETURNING log.id
postgres_1           | 2021-06-20 08:24:02.372 UTC [1228] ERROR:  relation "connection" does not exist at character 55
postgres_1           | 2021-06-20 08:24:02.372 UTC [1228] STATEMENT:  SELECT connection.conn_id AS connection_conn_id
postgres_1           | 	FROM connection GROUP BY connection.conn_id
postgres_1           | 	HAVING count(*) > 1
postgres_1           | 2021-06-20 08:24:02.374 UTC [1228] ERROR:  current transaction is aborted, commands ignored until end of transaction block
postgres_1           | 2021-06-20 08:24:02.374 UTC [1228] STATEMENT:  SELECT connection.password AS connection_password, connection.extra AS connection_extra, connection.id AS connection_id, connection.conn_id AS connection_conn_id, connection.conn_type AS connection_conn_type, connection.description AS connection_description, connection.host AS connection_host, connection.schema AS connection_schema, connection.login AS connection_login, connection.port AS connection_port, connection.is_encrypted AS connection_is_encrypted, connection.is_extra_encrypted AS connection_is_extra_encrypted
postgres_1           | 	FROM connection
postgres_1           | 	WHERE connection.conn_type IS NULL
redis_1              | 1:C 20 Jun 2021 06:00:18.328 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1              | 1:C 20 Jun 2021 06:00:18.328 # Redis version=6.2.4, bits=64, commit=00000000, modified=0, pid=1, just started
redis_1              | 1:C 20 Jun 2021 06:00:18.328 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1              | 1:M 20 Jun 2021 06:00:18.329 * monotonic clock: POSIX clock_gettime
redis_1              | 1:M 20 Jun 2021 06:00:18.329 * Running mode=standalone, port=6379.
redis_1              | 1:M 20 Jun 2021 06:00:18.329 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1              | 1:M 20 Jun 2021 06:00:18.329 # Server initialized
redis_1              | 1:M 20 Jun 2021 06:00:18.329 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1              | 1:M 20 Jun 2021 06:00:18.329 * Ready to accept connections
redis_1              | 1:M 20 Jun 2021 06:05:33.491 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.525 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.639 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.686 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.705 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.819 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:05:33.868 # Failed opening the RDB file crontab (in server root dir /etc) for saving: Permission denied
redis_1              | 1:C 20 Jun 2021 06:07:20.505 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1              | 1:C 20 Jun 2021 06:07:20.505 # Redis version=6.2.4, bits=64, commit=00000000, modified=0, pid=1, just started
redis_1              | 1:C 20 Jun 2021 06:07:20.505 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1              | 1:M 20 Jun 2021 06:07:20.506 * monotonic clock: POSIX clock_gettime
redis_1              | 1:M 20 Jun 2021 06:07:20.506 * Running mode=standalone, port=6379.
redis_1              | 1:M 20 Jun 2021 06:07:20.506 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1              | 1:M 20 Jun 2021 06:07:20.506 # Server initialized
redis_1              | 1:M 20 Jun 2021 06:07:20.506 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1              | 1:M 20 Jun 2021 06:07:20.507 * Ready to accept connections
redis_1              | 1:signal-handler (1624170928) Received SIGTERM scheduling shutdown...
redis_1              | 1:M 20 Jun 2021 06:35:28.766 # User requested shutdown...
redis_1              | 1:M 20 Jun 2021 06:35:28.766 * Saving the final RDB snapshot before exiting.
redis_1              | 1:M 20 Jun 2021 06:35:28.768 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 06:35:28.768 # Redis is now ready to exit, bye bye...
redis_1              | 1:C 20 Jun 2021 08:10:02.790 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1              | 1:C 20 Jun 2021 08:10:02.790 # Redis version=6.2.4, bits=64, commit=00000000, modified=0, pid=1, just started
redis_1              | 1:C 20 Jun 2021 08:10:02.790 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1              | 1:M 20 Jun 2021 08:10:02.791 * monotonic clock: POSIX clock_gettime
redis_1              | 1:M 20 Jun 2021 08:10:02.791 * Running mode=standalone, port=6379.
redis_1              | 1:M 20 Jun 2021 08:10:02.791 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1              | 1:M 20 Jun 2021 08:10:02.791 # Server initialized
redis_1              | 1:M 20 Jun 2021 08:10:02.791 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1              | 1:M 20 Jun 2021 08:10:02.820 * Loading RDB produced by version 6.2.4
redis_1              | 1:M 20 Jun 2021 08:10:02.820 * RDB age 5674 seconds
redis_1              | 1:M 20 Jun 2021 08:10:02.820 * RDB memory usage when created 0.77 Mb
redis_1              | 1:M 20 Jun 2021 08:10:02.820 * DB loaded from disk: 0.028 seconds
redis_1              | 1:M 20 Jun 2021 08:10:02.820 * Ready to accept connections
redis_1              | 1:M 20 Jun 2021 08:27:44.157 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.229 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.447 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.510 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.543 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.762 * DB saved on disk
redis_1              | 1:M 20 Jun 2021 08:27:44.855 # Failed opening the RDB file crontab (in server root dir /etc) for saving: Permission denied
airflow-init_1       | BACKEND=postgresql+psycopg2
airflow-init_1       | DB_HOST=postgres
airflow-init_1       | DB_PORT=5432
airflow-init_1       |
airflow-init_1       | DB: postgresql+psycopg2://airflow:***@postgres/airflow
airflow-init_1       | [2021-06-20 08:28:11,323] {db.py:695} INFO - Creating tables
airflow-init_1       | INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
airflow-init_1       | INFO  [alembic.runtime.migration] Will assume transactional DDL.
airflow-webserver_1  | BACKEND=postgresql+psycopg2
airflow-webserver_1  | DB_HOST=postgres
airflow-webserver_1  | DB_PORT=5432
airflow-webserver_1  |
airflow-worker_1     | BACKEND=postgresql+psycopg2
airflow-worker_1     | DB_HOST=postgres
airflow-worker_1     | DB_PORT=5432
flower_1             | BACKEND=postgresql+psycopg2
flower_1             | DB_HOST=postgres
flower_1             | DB_PORT=5432
airflow-worker_1     |
airflow-worker_1     | BACKEND=postgresql+psycopg2
airflow-worker_1     | DB_HOST=postgres
airflow-worker_1     | DB_PORT=5432
flower_1             |
flower_1             | BACKEND=postgresql+psycopg2
flower_1             | DB_HOST=postgres
flower_1             | DB_PORT=5432
airflow-worker_1     |
flower_1             |
airflow-scheduler_1  | BACKEND=postgresql+psycopg2
airflow-scheduler_1  | DB_HOST=postgres
airflow-scheduler_1  | DB_PORT=5432
airflow-scheduler_1  |
airflow-scheduler_1  | BACKEND=postgresql+psycopg2
airflow-scheduler_1  | DB_HOST=postgres
airflow-scheduler_1  | DB_PORT=5432
airflow-scheduler_1  |
airflow-init_1       | Upgrades done
airflow-scheduler_1  |   ____________       _____________
airflow-scheduler_1  |  ____    |__( )_________  __/__  /________      __
airflow-scheduler_1  | ____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
airflow-scheduler_1  | ___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
airflow-scheduler_1  |  _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
airflow-scheduler_1  | [2021-06-20 08:28:24,667] {scheduler_job.py:1253} INFO - Starting the scheduler
airflow-scheduler_1  | [2021-06-20 08:28:24,667] {scheduler_job.py:1258} INFO - Processing each file at most -1 times
airflow-scheduler_1  | [2021-06-20 08:28:24,703] {dag_processing.py:254} INFO - Launched DagFileProcessorManager with pid: 28
airflow-scheduler_1  | [2021-06-20 08:28:24,721] {scheduler_job.py:1822} INFO - Resetting orphaned tasks for active dag runs
airflow-scheduler_1  | [2021-06-20 08:28:24,770] {settings.py:52} INFO - Configured default timezone Timezone('UTC')
flower_1             | [2021-06-20 08:28:26,062] {command.py:137} INFO - Visit me at http://0.0.0.0:5555
flower_1             | [2021-06-20 08:28:26,175] {command.py:142} INFO - Broker: redis://redis:6379/0
flower_1             | [2021-06-20 08:28:26,193] {command.py:145} INFO - Registered tasks:
flower_1             | ['airflow.executors.celery_executor.execute_command',
flower_1             |  'celery.accumulate',
flower_1             |  'celery.backend_cleanup',
flower_1             |  'celery.chain',
flower_1             |  'celery.chord',
flower_1             |  'celery.chord_unlock',
flower_1             |  'celery.chunks',
flower_1             |  'celery.group',
flower_1             |  'celery.map',
flower_1             |  'celery.starmap']
flower_1             | [2021-06-20 08:28:26,587] {mixins.py:229} INFO - Connected to redis://redis:6379/0
airflow-worker_1     | Starting flask
airflow-worker_1     |  * Serving Flask app "airflow.utils.serve_logs" (lazy loading)
airflow-worker_1     |  * Environment: production
airflow-worker_1     |    WARNING: This is a development server. Do not use it in a production deployment.
airflow-worker_1     |    Use a production WSGI server instead.
airflow-worker_1     |  * Debug mode: off
airflow-worker_1     | [2021-06-20 08:28:27,201] {_internal.py:113} INFO -  * Running on http://0.0.0.0:8793/ (Press CTRL+C to quit)
airflow-worker_1     | /home/airflow/.local/lib/python3.6/site-packages/celery/platforms.py:801 RuntimeWarning: You're running the worker with superuser privileges: this is
airflow-worker_1     | absolutely not recommended!
airflow-worker_1     |
airflow-worker_1     | Please specify a different user using the --uid option.
airflow-worker_1     |
airflow-worker_1     | User information: uid=0 euid=0 gid=0 egid=0
airflow-worker_1     |
flower_1             | [2021-06-20 08:28:28,040] {inspector.py:42} WARNING - Inspect method stats failed
flower_1             | [2021-06-20 08:28:28,043] {inspector.py:42} WARNING - Inspect method scheduled failed
flower_1             | [2021-06-20 08:28:28,049] {inspector.py:42} WARNING - Inspect method registered failed
flower_1             | [2021-06-20 08:28:28,050] {inspector.py:42} WARNING - Inspect method active failed
flower_1             | [2021-06-20 08:28:28,051] {inspector.py:42} WARNING - Inspect method active_queues failed
flower_1             | [2021-06-20 08:28:28,056] {inspector.py:42} WARNING - Inspect method conf failed
flower_1             | [2021-06-20 08:28:28,056] {inspector.py:42} WARNING - Inspect method revoked failed
flower_1             | [2021-06-20 08:28:28,058] {inspector.py:42} WARNING - Inspect method reserved failed
airflow-worker_1     | [2021-06-20 08:28:33,369: INFO/MainProcess] Connected to redis://redis:6379/0
airflow-worker_1     | [2021-06-20 08:28:33,385: INFO/MainProcess] mingle: searching for neighbors
airflow-worker_1     | [2021-06-20 08:28:34,415: INFO/MainProcess] mingle: all alone
airflow-worker_1     | [2021-06-20 08:28:34,496: INFO/MainProcess] celery@d8a26078ed1d ready.
airflow-init_1       | airflow already exist in the db
airflow-worker_1     | [2021-06-20 08:28:36,212: INFO/MainProcess] Events of group {task} enabled by remote.
airflow-webserver_1  |   ____________       _____________
airflow-webserver_1  |  ____    |__( )_________  __/__  /________      __
airflow-webserver_1  | ____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
airflow-webserver_1  | ___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
airflow-webserver_1  |  _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
airflow-webserver_1  | [2021-06-20 08:28:37,237] {dagbag.py:487} INFO - Filling up the DagBag from /dev/null
airflow-init_1       | 2.1.0
d_airflow-init_1 exited with code 0
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [47] [INFO] Starting gunicorn 20.1.0
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [47] [INFO] Listening at: http://0.0.0.0:8080 (47)
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [47] [INFO] Using worker: sync
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [51] [INFO] Booting worker with pid: 51
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [52] [INFO] Booting worker with pid: 52
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [53] [INFO] Booting worker with pid: 53
airflow-webserver_1  | [2021-06-20 08:28:43 +0000] [54] [INFO] Booting worker with pid: 54
```



##### 6. 管理后台

> Browser : http://0.0.0.0:8080
>
> Username: airflow
>
> Password: airflow

**Web admin**

<img src="./imgs/image-20210620163251130.png" alt="image-20210620163251130" style="zoom:50%;" />

##### 

**Dashboard**

<img src="./imgs/image-20210620145955347.png" alt="image-20210620145955347" style="zoom:50%;" />

**Flower**

> Browser: http://139.198.26.17:5555

<img src="./imgs/image-20210620163935053.png" alt="image-20210620163935053" style="zoom:50%;" />

##### 7. 访问

*如果在容器内部署，那么如何在容器外访问容器内的环境，该如何操作呢？*



###### 方式1

```bash
# 首先切换到 docker-compose.yaml 同一级目录
cd deploy

# 查看详情
docker-compose run airflow-worker info

# 进入Python环境
docker-compose run airflow-worker python
```

###### 方式2

```bash
# 首先切换到 docker-compose.yaml 同一级目录
cd deploy

# 其次下载 airflow.sh 脚本
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.1.0/airflow.sh'
chmod +x airflow.sh

# 执行命令
./airflow.sh info

./airflow.sh bash

./airflow.sh python
```

##### 8. 卸载

```bash
# 要停止和删除容器、删除包含数据库数据的卷并下载图像
docker-compose down --volumes --rmi all
```



#### 3. 注意事项

##### 1. 本地部署

**pip 版本**

```bash
# https://github.com/apache/airflow/issues/12838
# pip和airflow 2.x兼容性问题，要降级pip的版本，否则在通过pip安装时有异常
pip3.7 install --upgrade pip==20.2.4
```

**pip.conf**

pip 建议用 *mirrors.aliyun.com* 阿里云的源头，如果用 豆瓣的 *mirrors.douban.com* 的会出现安装时版本依赖的问题。

**https://raw.githubusercontent.com/ 网络不通**

```bash
# For example: https://raw.githubusercontent.com/apache/airflow/constraints-2.1.0/constraints-3.6.txt # 有可能网络不通，见<<3. 注意事项>>
pip3.7 install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# 如果上个命令网络不通，导致无法反问 raw.githubusercontent.com 可以替换以下命令
pip3.7 install "apache-airflow==${AIRFLOW_VERSION}" --constraint https://zhengyansheng.oss-cn-beijing.aliyuncs.com/constraints-3.7.txt
```

##### 2. Docker部署

```bash
# https://github.com/apache/airflow/issues/15312
# docker-compose.yaml默认AIRFLOW_IMAGE_NAME是变量，需要在外部指定版本镜像和版本才行，否则初始化时异常抱错。
export AIRFLOW_IMAGE_NAME=apache/airflow:2.1.0  
```



