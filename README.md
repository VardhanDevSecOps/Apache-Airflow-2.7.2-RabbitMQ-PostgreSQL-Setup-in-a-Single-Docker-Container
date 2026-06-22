```
+------------------------------------------------+
| Master Container                               |
|------------------------------------------------|
| PostgreSQL                                     |
| RabbitMQ                                       |
| Airflow Scheduler                              |
| Airflow Webserver                              |
+-------------------+----------------------------+
                    |
                    |
         RabbitMQ Broker
                    |
      +-------------+--------------+
      |                            |
      |                            |
+-----v------+             +-------v------+
| Worker-1   |             | Worker-2     |
| Celery     |             | Celery       |
| Worker     |             | Worker       |
+------------+             +--------------+
```
------------------------------------------------------------------------------

Deploy the following components inside a single Docker container:

◉ Apache Airflow 2.7.2

◉ RabbitMQ

◉ PostgreSQL

◉ Airflow Webserver

◉ Airflow Scheduler

◉ Airflow Celery Worker

◉ Airflow Triggerer

Access externally via:

| Service                | Port  |
| ---------------------- | ----- |
| Airflow UI             | 8080  |
| RabbitMQ AMQP          | 5672  |
| RabbitMQ Management UI | 15672 |
| PostgreSQL             | 5432  |

------------------------------------------------------------------------------

Prerequisites

**Verify Docker installation**
```
docker --version
```
**Start Docker:**
```
sudo systemctl start docker
```
**Verify Docker daemon:**
```
docker ps
```
---------------------------------------------------------------------------------------------------------------------------
**Step 1: Create Ubuntu Network**

```
docker network create airflow-net
```
Verify:

```
docker network ls
```
<img width="568" height="180" alt="Screenshot 2026-06-22 at 4 21 40 PM" src="https://github.com/user-attachments/assets/e5141251-8de0-4c76-b603-8f7682b93e0b" />


-----------------------------------------------------------------------------------------------------------------------------------------

**Step 2: Create Master Container**
Pull Ubuntu:
```
docker pull ubuntu:22.04
```
<img width="1087" height="262" alt="Screenshot 2026-06-22 at 4 29 13 PM" src="https://github.com/user-attachments/assets/936f32db-cb5c-44ba-bf23-c3e919bab821" />

Create container:
```
docker run -dit \
--name airflow-master \
--hostname airflow-master \
--network airflow-net \
-p 8080:8080 \
-p 5432:5432 \
-p 5672:5672 \
-p 15672:15672 \
ubuntu:22.04
```
<img width="1085" height="340" alt="Screenshot 2026-06-22 at 4 32 21 PM" src="https://github.com/user-attachments/assets/8b7b4ef4-2f15-4b21-9550-fa644fe926a7" />

Login:
```
docker exec -it airflow-master bash
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 3: Install Dependencies**

Inside master container:
```
apt update
```
```
apt install -y \
python3 \
python3-pip \
python3-venv \
postgresql \
postgresql-contrib \
rabbitmq-server \
curl \
vim \
net-tools
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 4: Create Airflow Virtual Environment**
```
python3 -m venv /opt/airflow_venv
```
```
source /opt/airflow_venv/bin/activate
```
Upgrade pip:
```
pip install --upgrade pip
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 5: Install Airflow 2.7.2**
```
AIRFLOW_VERSION=2.7.2
PYTHON_VERSION=3.10
```
```
pip install \
"apache-airflow[celery,postgres,rabbitmq]==2.7.2" \
--constraint \
https://raw.githubusercontent.com/apache/airflow/constraints-2.7.2/constraints-3.10.txt
```
Verify:
```
airflow version
```
Expected:
```
2.7.2
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 6: Start PostgreSQL**
```
service postgresql start
```
Create DB:
```
su - postgres
```
```
psql
```
Inside postgres:
```
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
\q
exit
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 7: Start RabbitMQ**
```
service rabbitmq-server start
```
Enable management:
```
rabbitmq-plugins enable rabbitmq_management
```
Check:
```
rabbitmqctl status
```
Management UI:
```
http://localhost:15672
```
Default:
```
guest or airflow or airflow
guest or airflow or airflow123
```
After successful execution Rabbitmq will show below

<img width="1580" height="324" alt="Screenshot 2026-06-22 at 6 20 50 PM" src="https://github.com/user-attachments/assets/82e6a4a8-506f-41cc-b162-72f0b771d3da" />


-----------------------------------------------------------------------------------------------------------------------------------------
**Step 8: Initialize Airflow**
Set home:
```
export AIRFLOW_HOME=/opt/airflow
```
Create:
```
mkdir -p $AIRFLOW_HOME
```
Initialize:
```
airflow db init
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 9: Configure Airflow**
Edit:
```
vim $AIRFLOW_HOME/airflow.cfg
```
Change:

Executor
```
executor = CeleryExecutor
```
SQLAlchemy
```
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@localhost:5432/airflow
```
Broker
```
broker_url = amqp://guest:guest@localhost:5672//
```
Result Backend
```
result_backend = db+postgresql://airflow:airflow@localhost:5432/airflow
```
Load Examples
```
load_examples = False
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 10: Create Admin User**
```
airflow users create \
--username admin \
--password admin \
--firstname Admin \
--lastname User \
--role Admin \
--email admin@test.com
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 11: Start Scheduler**

Open terminal inside container:
```
source /opt/airflow_venv/bin/activate
```
```
export AIRFLOW_HOME=/opt/airflow
```
```
airflow scheduler
```
Leave running.

-----------------------------------------------------------------------------------------------------------------------------------------

**Error's for airflow scheduler not wokring**

If you found the ERROR - Exception when running scheduler job Traceback (most recent call last), go for below soultion.

Verify Current Database Configuration

Run:
```
airflow config get-value database sql_alchemy_conn
```
If you see something like:
```
sqlite:////home/postgres/airflow/airflow.db
```
or
```
sqlite:////root/airflow/airflow.db
```
then Airflow is still using SQLite.

If using SQLite temporarily:
```
sudo chown -R postgres:postgres $AIRFLOW_HOME
chmod -R 755 $AIRFLOW_HOME
chmod 664 $AIRFLOW_HOME/airflow.db
```
Then:
```
airflow db migrate
airflow scheduler
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 12: Start Webserver**

New terminal:
```
docker exec -it airflow-master bash
```
```
source /opt/airflow_venv/bin/activate
```
```
export AIRFLOW_HOME=/opt/airflow
```
```
airflow webserver --port 8080
```
After successful below UI display & Password admin, admin123

<img width="1662" height="447" alt="Screenshot 2026-06-22 at 7 31 31 PM" src="https://github.com/user-attachments/assets/91df4baa-f60e-406d-b71c-4d1685c8431d" />

<img width="1674" height="454" alt="Screenshot 2026-06-22 at 7 45 54 PM" src="https://github.com/user-attachments/assets/91da0828-5a8f-44c5-ac31-d5761a992d34" />

In case any login issue, delete admin user, which you created earlier & re-create it with below

Reset the admin password:
```
airflow users reset-password \
  --username admin \
  --password admin123
```
If your Airflow version doesn't support reset-password, run:
```
airflow users delete --username admin
```
Then recreate:
```
airflow users create \
  --username admin \
  --password admin123 \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@test.com
  ```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 13: Create Worker-1 Container**
```
docker run -dit \
--name worker1 \
--hostname worker1 \
--network airflow-net \
ubuntu:22.04
```
Login:
```
docker exec -it worker1 bash
```
Install:
```
apt update
apt install -y \
python3 \
python3-pip \
python3-venv
```
Create venv:
```
python3 -m venv /opt/airflow_venv
source /opt/airflow_venv/bin/activate
```
Install Airflow:
```
pip install \
"apache-airflow[celery,postgres,rabbitmq]==2.7.2" \
--constraint \
https://raw.githubusercontent.com/apache/airflow/constraints-2.7.2/constraints-3.10.txt
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 14: Configure Worker-1**
```
export AIRFLOW_HOME=/opt/airflow
mkdir -p $AIRFLOW_HOME
```
Generate config:
```
airflow db init
```
Edit:
```
vim $AIRFLOW_HOME/airflow.cfg
```
Set:
```
executor = CeleryExecutor
broker_url = amqp://guest:guest@airflow-master:5672//
result_backend = db+postgresql://airflow:airflow@airflow-master:5432/airflow
```
if any issues happen while deploying like Time zone mismatch follow below commands

Fix

Install timezone package:
```
apt update
apt install -y tzdata
```
Configure timezone:
```
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
echo "Asia/Kolkata" > /etc/timezone
```
Then export:
```
export TZ=Asia/Kolkata
```
Add permanently:
```
echo "export TZ=Asia/Kolkata" >> ~/.bashrc
source ~/.bashrc
```
Test:
```
python3 -c "import pendulum; print(pendulum.now())"
```
-----------------------------------------------------------------------------------------------------------------------------------------






