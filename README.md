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

**If login issues happen means use below commands for access Rabbitmq**

**Create a New Admin User**

Run the following commands:
```
rabbitmqctl add_user airflow airflow123
```

Give administrator permissions:
```
rabbitmqctl set_user_tags airflow administrator
```

Give full permissions:
```
rabbitmqctl set_permissions -p / airflow ".*" ".*" ".*"
```
List users:
```
rabbitmqctl list_users
```

Expected output:
```
Listing users ...
user      tags
guest     [administrator]
airflow   [administrator]
```
Now log in using:
```
Username: airflow
Password: airflow123
```
-----------------------------------------------------------------------------------------------------------------------------------------
**Step 8: Initialize ƒ**
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

broker_url = amqp://airflow:airflow123@airflow-master:5672//

result_backend = db+postgresql://airflow:airflow@airflow-master:5432/airflow

sql_alchemy_conn = postgresql+psycopg2://user:password@airflow-master:5432/airflow
```
or export:
```
export AIRFLOW__CELERY__BROKER_URL='amqp://airflow:airflow123@airflow-master:5672//
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
<img width="927" height="567" alt="Screenshot 2026-06-22 at 8 22 15 PM" src="https://github.com/user-attachments/assets/c7ae08e4-a74d-4e57-91c9-7c2c3de7d82d" />

-----------------------------------------------------------------------------------------------------------------------------------------

**While deploying if you facing any issue, like connection refused or Rabbitmq logins, go with below commands**

Step 1: Check PostgreSQL on Master

On the Airflow Master container/server:
```
ps -ef | grep postgres
```
or
```
systemctl status postgresql
```
If running in Docker:
```
docker ps
```
Look for a PostgreSQL container.

**Step 2: Verify Port 5432 is Listening**

On Airflow Master:
```
ss -tulpn | grep 5432
```
Expected:
```
tcp LISTEN 0 244 0.0.0.0:5432
```
or
```
tcp LISTEN 0 244 127.0.0.1:5432
```
If nothing appears, PostgreSQL is not running.

**Step 3: Test from Worker**

On Worker1:
```
nc -zv airflow-master 5432
```
or
```
telnet airflow-master 5432
```
Expected:
```
Connection to airflow-master 5432 port [tcp/postgresql] succeeded
```
If you get:

Connection refused

PostgreSQL is down.

If you get:

No route to host

Docker networking issue.

**Step 4: Check Your Airflow DB Connection**

On Worker1:
```
airflow config get-value database sql_alchemy_conn
```
It should look similar to:
```
postgresql+psycopg2://airflow:airflow@airflow-master:5432/airflow
```
**Step 5: Verify PostgreSQL Container Name**

If using Docker:
```
docker ps -a
```
Example:
```
CONTAINER ID   IMAGE        NAMES
abc123         postgres     postgres
def456         airflow      airflow-master
```
If PostgreSQL is in a separate container called postgres, then your Airflow connection should be:
```
postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
```
NOT
```
postgresql+psycopg2://airflow:airflow@airflow-master:5432/airflow
```
unless PostgreSQL is actually running inside the airflow-master container.

On Master
```
docker ps -a
ss -tulpn | grep 5432
```
On Worker
```
airflow config get-value database sql_alchemy_conn
nc -zv airflow-master 5432
```
Still you facing the issue with the command - ss -tulpn | grep 5432

On the master and worker:
```
apt update
apt install -y net-tools netcat-openbsd telnet
```
Then run:
```
netstat -tulpn | grep 5432
Check if PostgreSQL is running on airflow-master
```
**On the master node:**
```
ps -ef | grep postgres
```

If PostgreSQL is running, you'll see several postgres processes.
If nothing appears, PostgreSQL is not running.

If PostgreSQL is installed locally on airflow-master

Check status:

```
service postgresql status
```

or
```
systemctl status postgresql
```

Start it if needed:

```
service postgresql start
```
Test connectivity from worker

After installing netcat:

```
nc -zv airflow-master 5432
```

Expected:

```
Connection to airflow-master 5432 port [tcp/postgresql] succeeded
```

Most important command now

On airflow-master,

```
ps -ef | grep postgres
```
and

```
netstat -tulpn | grep 5432
```

If netstat is missing:

```
apt install -y net-tools
```

Check on Master cluster

```
ps -ef | grep postgres
```

If it's working fine ok or go with below commands

**Step 6: Edit postgresql.conf**

On airflow-master:

```
vi /etc/postgresql/14/main/postgresql.conf
```

Find:

```
#listen_addresses = 'localhost'
```

Change to:

```
listen_addresses = '*'
```

Save.

**Step 7: Edit pg_hba.conf**

Open:

```
vi /etc/postgresql/14/main/pg_hba.conf
```

Add this line at the bottom:

```
host    all     all     172.22.0.0/16     md5
```

Since your Docker network is:

```
airflow-master = 172.22.0.2
```

This allows workers on that network to connect.

For testing you can temporarily use:

```
host    all     all     0.0.0.0/0     md5
```

(Only for lab environments.)

**Step 8: Restart PostgreSQL**

```
service postgresql restart
```

Verify:

```
netstat -tulpn | grep 5432
```

Expected:

```
tcp 0 0 0.0.0.0:5432 0.0.0.0:* LISTEN
```

or

```
tcp6 0 0 :::5432 :::* LISTEN
```

**Step 9: Test From Worker**

On Worker1:

```
nc -zv airflow-master 5432
```

Expected:

```
Connection to airflow-master 5432 port [tcp/postgresql] succeeded
```

**Step 10: Test PostgreSQL Login**

On Worker1 install client:

```
apt update
apt install -y postgresql-client
```

Then:

```
psql -h airflow-master \
     -U airflow \
     -d airflow
```

If prompted:

```
export PGPASSWORD=airflow
```

Then retry.

**Step 11: Start Worker**

Once PostgreSQL connectivity works:

```
airflow celery worker
```

The worker should register successfully.

If still not able to login with Rabbitmq server, Enter the below commands

**On airflow-master**

Check RabbitMQ users:

```
rabbitmqctl list_users
```

You will likely see:

```
guest   [administrator]
```

Create Airflow User

```
rabbitmqctl add_user airflow airflow123
```

Grant permissions:

```
rabbitmqctl set_user_tags airflow administrator
```

rabbitmqctl set_permissions -p / airflow ".*" ".*" ".*"

Verify:

```
rabbitmqctl list_users
```

Expected:

```
guest
airflow
Test RabbitMQ Login
```

On worker:

```
telnet airflow-master 5672
```

Should connect.

Update Airflow Broker URL

Check current value:

```
airflow config get-value celery broker_url
```

You probably have:

```
amqp://guest:guest@airflow-master:5672//
```
Change it in airflow.cfg:
```

vi ~/airflow/airflow.cfg
```

Find:

```
broker_url = amqp://guest:guest@airflow-master:5672//
```

Replace:

```
broker_url = amqp://airflow:airflow123@airflow-master:5672//
```

Or export:

```
export AIRFLOW__CELERY__BROKER_URL='amqp://airflow:airflow123@airflow-master:5672//'
```

Verify RabbitMQ

On master:

```
rabbitmqctl list_connections
```

After worker starts you should see an active connection.

Start Worker Again

```
airflow celery worker
```

Expected:

```
celery@worker1 ready
```

The Rabbitmq Dashboard looks like this

<img width="1649" height="755" alt="Screenshot 2026-06-23 at 1 42 34 PM" src="https://github.com/user-attachments/assets/da8a27c4-4c2a-4398-83a3-76be15d1b875" />

With worke1 

<img width="1660" height="552" alt="Screenshot 2026-06-23 at 1 44 12 PM" src="https://github.com/user-attachments/assets/0ea6b0f3-7ca9-43cb-bef0-a6b4ce9cc33b" />

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

**You can also run the native Celery command on a machine that has access to the broker:**

```
celery -A airflow.providers.celery.executors.celery_executor.app status
```
or

```
celery -A airflow.providers.celery.executors.celery_executor.app inspect ping
```

This shows all active workers.

<img width="1026" height="224" alt="Screenshot 2026-06-24 at 7 18 29 PM" src="https://github.com/user-attachments/assets/9f3b31c0-0273-4ce6-b4f4-fe36e780d339" />

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

**Create the 2nd worker container**

From terminal:

```
docker run -dit \
--name worker2 \
--hostname worker2 \
--network airflow-net \
ubuntu:22.04
```
Repeat the same installation steps as Worker1.

Start:

```
airflow celery worker
```
-------------------------------------------------------------------------------------------------------------------------------------------------------------------


<img width="1667" height="439" alt="Screenshot 2026-06-25 at 11 46 04 AM" src="https://github.com/user-attachments/assets/461a92b7-2645-48be-9f4c-33a8555d7071" />





















