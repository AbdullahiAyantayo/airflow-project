## Introduction to Airflow DAGs
In Airflow, data pipelines are defined in Python code as directed acyclic graphs, also known as DAGs. Within a graph, each node represents a task which completes a unit of work, and each edge represents a dependency between tasks.


What is a DAG?
In Airflow, a directed acyclic graph (DAG) is a data pipeline defined in Python code. Each DAG represents a collection of tasks you want to run and is organized to show relationships between tasks in the Airflow UI. The mathematical properties of DAGs make them useful for building data pipelines:

Directed: If multiple tasks exist, then each task must have at least one defined upstream or downstream task.
Acyclic: Tasks cannot have a dependency to themselves. This avoids infinite loops.
Graph: All tasks can be visualized in a graph structure, with relationships between tasks defined by nodes and vertices.
Aside from these requirements, DAGs in Airflow can be defined however you need! They can have a single task or thousands of tasks arranged in any number of ways.

An instance of a DAG running on a specific date is called a DAG run. DAG runs can be started by the Airflow scheduler based on the DAG's defined schedule, or they can be started manually.

Writing a DAG
DAGs in Airflow are defined in a Python script that is placed in an Airflow project's DAG_FOLDER. Airflow will execute the code in this folder to load any DAG objects. If you are working with the Astro CLI, DAG code is placed in the dags folder.

Most DAGs follow this general flow within a Python script:

Imports: Any needed Python packages are imported at the top of the DAG script. This always includes a dag function import (either the DAG class or the dag decorator). It can also include provider packages or other general Python packages.
DAG instantiation: A DAG object is created and any DAG-level parameters such as the schedule interval are set.
Task instantiation: Each task is defined by calling an operator and providing necessary task-level parameters.
Dependencies: Any dependencies between tasks are set using bitshift operators (<< and >>), the set_upstream() or set_downstream functions, or the chain() function. Note that if you are using the TaskFlow API, dependencies are inferred based on the task function calls.
The following example DAG loads data from Amazon S3 to Snowflake, runs a Snowflake query, and then sends an email.

from airflow import DAG
from airflow.operators.email import EmailOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.snowflake.transfers.s3_to_snowflake import S3ToSnowflakeOperator

from pendulum import datetime, duration

# Instantiate DAG
with DAG(
    dag_id="s3_to_snowflake",
    start_date=datetime(2023, 1, 1),
    schedule="@daily",
    default_args={"retries": 1, "retry_delay": duration(minutes=5)},
    catchup=False,
):
    # Instantiate tasks within the DAG context
    load_file = S3ToSnowflakeOperator(
        task_id="load_file",
        s3_keys=["key_name.csv"],
        stage="snowflake_stage",
        table="my_table",
        schema="my_schema",
        file_format="csv",
        snowflake_conn_id="snowflake_default",
    )

    snowflake_query = SnowflakeOperator(
        task_id="run_query", sql="SELECT COUNT(*) FROM my_table"
    )

    send_email = EmailOperator(
        task_id="send_email",
        to="noreply@astronomer.io",
        subject="Snowflake DAG",
        html_content="<p>The Snowflake DAG completed successfully.<p>",
    )

    # Define dependencies
    load_file >> snowflake_query >> send_email
