# My First Data Engineering Pipeline with Apache Airflow

**Date:** June 2025  
**Author:** Sofia Calderon

---

I’ve spent most of my academic and professional journey working on data science and analytics. From exploring socio-economic datasets in R to deploying dashboards in ArcGIS and Looker Studio, I’ve always loved the process of turning raw data into actionable insights.

But over time, a recurring question kept nudging at me:

> **How can I make these workflows more automated, scalable, and production-ready?**

That curiosity led me into the world of **data engineering**—and more specifically, into **Apache Airflow**.

---

## 🧠 Bridging Software Engineering with Data

While I’ve spent years working with data, my Python training has mostly been rooted in **software engineering**. I’ve learned how to write clean, modular code, follow best practices, use version control, and design systems that scale.

When I found **data engineering**, it felt like the perfect blend:  
It lets me use all the software engineering skills I’ve built while working with data pipelines, automation, APIs, and cloud infrastructure. It was exactly the bridge between my two interests.

---

## 🛠️ Getting Started: Docker, VS Code & a Solid Course

To get hands-on experience, I set up my environment using:

- **Docker**: to create isolated containers for Airflow, Postgres, and supporting services. This setup made everything reproducible and easy to reset.
- **Visual Studio Code**: my primary IDE for writing DAGs and managing the project structure.
- **A Udemy course** to guide me through the process:  
  🎓 [*The Complete Hands-On Course to Master Apache Airflow*](https://www.udemy.com/course/the-complete-hands-on-course-to-master-apache-airflow/) by Marc Lamberti.

The course is incredibly thorough and practical—it helped me understand how real-world data pipelines are structured, and gave me confidence to build something of my own.

---

## 💡 What I Built: A Simple ETL DAG

To apply what I was learning, I created a DAG that:

1. Creates a `users` table in a PostgreSQL database.
2. Uses a **sensor** to check whether a public API is available.
3. Extracts fake user data from that API.
4. Saves the data as a CSV file in a temporary directory.
5. Loads the data into the database.

It’s not flashy, but it’s my **first real data pipeline**, and I’m proud of it.

---

## 🔍 Code Walkthrough

Below is the full DAG that I created using Airflow's SDK:

```python
from airflow.sdk import dag, task
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.sdk.bases.sensor import PokeReturnValue
from airflow.providers.postgres.hooks.postgres import PostgresHook

@dag
def user_processing():

    create_table = SQLExecuteQueryOperator(
        task_id="create_table",
        conn_id="postgres",
        sql=""" 
        CREATE TABLE IF NOT EXISTS users (
            id INT PRIMARY KEY, 
            firstName VARCHAR(255), 
            lastName VARCHAR(255), 
            email VARCHAR(255), 
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)
        """,
    )

    @task.sensor(poke_interval=5, timeout=300)
    def is_api_availible() -> PokeReturnValue:
        import requests
        response = requests.get(
            "https://raw.githubusercontent.com/marclamberti/datasets/refs/heads/main/fakeuser.json"
        )
        if response.status_code == 200:
            return PokeReturnValue(is_done=True, xcom_value=response.json())
        return PokeReturnValue(is_done=False)

    @task
    def extract_user(fake_user):
        return {
            "id": fake_user["id"],
            "firstname": fake_user["personalInfo"]["firstName"],
            "lastname": fake_user["personalInfo"]["lastName"],
            "email": fake_user["personalInfo"]["email"],
        }

    @task
    def process_user(user_info):
        import csv
        from datetime import datetime
        user_info["created_at"] = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        with open("/tmp/user_info.csv", "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=user_info.keys())
            writer.writeheader()
            writer.writerow(user_info)

    @task
    def store_user():
        hook = PostgresHook(postgres_conn_id="postgres")
        hook.copy_expert(
            sql="COPY users FROM STDIN WITH CSV HEADER",
            filename="/tmp/user_info.csv"
        )

    process_user(extract_user(create_table >> is_api_availible())) >> store_user()

user_processing()
version: "3"
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    ports:
      - "5432:5432"

  webserver:
    image: apache/airflow:2.9.0
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    volumes:
      - ./dags:/opt/airflow/dags
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```

I kept my code in the ./dags folder and launched the services using:

docker compose up

From there, I could view the Airflow UI at localhost:8080 and watch my DAGs run!

##🔄 What I Learned
- Here are a few takeaways from building my first Airflow pipeline:
- Writing DAGs is just writing Python — which made things easier coming from a CS background.
- Sensors are powerful — you can wait for conditions like an API being available.
- Docker is your best friend — I could reset everything with a single command.
- Airflow encourages clean architecture — breaking tasks into small, testable functions.

## 🚀 What’s Next

- ✅ Build a more advanced DAG with branching logic.  
- ☁️ Integrate with Google Cloud (BigQuery + Cloud Storage).  
- 🔁 Explore `dbt` and `Kafka` for batch and stream processing.  
- 🐳 Keep refining my Docker and deployment skills.

