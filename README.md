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
-----------------------------------------------------------------------------------------------------------------------------------------












