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
