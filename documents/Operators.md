## Airflow operators
Operators are the building blocks of Airflow DAGs. They contain the logic of how data is processed in a pipeline. Each task in a DAG is defined by instantiating an operator.

There are many different types of operators available in Airflow. Some operators such as Python functions execute general code provided by the user, while other operators perform very specific actions such as transferring data from one system to another.

In this guide, you'll learn the basics of using operators in Airflow and then implement them in a DAG.


## What are Operators?
Operators are Python classes that encapsulate logic to do a unit of work. They can be viewed as a wrapper around each unit of work that defines the actions that will be completed and abstract the majority of code you would typically need to write. When you create an instance of an operator in a DAG and provide it with its required parameters, it becomes a task.

All operators inherit from the abstract BaseOperator class, which contains the logic to execute the work of the operator within the context of a DAG.

The following are some of the most frequently used Airflow operators:

PythonOperator: Executes a Python function.
BashOperator: Executes a bash script.
KubernetesPodOperator: Executes a task defined as a Docker image in a Kubernetes Pod.
SnowflakeOperator: Executes a query against a Snowflake database.
Operators typically only require a few parameters. Keep the following considerations in mind when using Airflow operators:


The core Airflow package includes basic operators such as the PythonOperator and BashOperator. These operators are automatically available in your Airflow environment. All other operators are part of provider packages, which you must install separately. For example, the SnowflakeOperator is part of the Snowflake provider package.
If an operator exists for your specific use case, you should use it instead of your own Python functions or hooks. This makes your DAGs easier to read and maintain.
If an operator doesn't exist for your use case, you can extend an operator to meet your needs. For more information about customizing operators, see the video Anatomy of an Operator.
Sensors are a type of operator that waits for something to happen. They can be used to make your DAGs more event-driven.
Deferrable Operators are a type of operator that releases their worker slot while waiting for their work to be completed. This can result in cost savings and greater scalability. Airflow recommends using deferrable operators whenever one exists for your use case and your task takes longer than a minute. You must be using Airflow 2.2 or later and have a triggerer running to use deferrable operators.
Any operator that interacts with a service external to Airflow typically requires a connection so that Airflow can authenticate to that external system. For more information about setting up connections, see Managing your connections in Apache Airflow or in the examples to follow.
Example implementation
The following example shows how to use multiple operators in a DAG to transfer data from Amazon S3 to Redshift and perform data quality checks.



The following operators are used in this example:

EmptyOperator: Organizes the flow of tasks in the DAG. Included with Airflow.
PythonDecoratedOperator: Executes a Python function. It is functionally the same as the PythonOperator, but it is instantiated using the @task decorator. Included with Airflow.
LocalFilesystemToS3Operator: Uploads a file from a local file system to Amazon S3. Included with the AWS provider package.
S3ToRedshiftOperator: Transfers data from Amazon S3 to Redshift. Included with the AWS provider package.
PostgresOperator: Executes a query against a Postgres database. Included with the Postgres provider package.
SQLCheckOperator: Checks against a database using a SQL query. Included with Airflow.
There are a few things to note about the operators in this example DAG:

Every operator is given a task_id. This is a required parameter, and the value provided is displayed as the name of the task in the Airflow UI.
Each operator requires different parameters based on the work it does. For example, the PostgresOperator has a sql parameter for the SQL script to be executed, and the S3ToRedshiftOperator has parameters to define the location and keys of the files being copied from Amazon S3 and the Redshift table receiving the data.
Connections to external systems are passed in most of these operators. The parameters conn_id, postgres_conn_id, and aws_conn_id all point to the names of the relevant connections stored in Airflow.
The following code shows how each of the operators is instantiated in a DAG file to define the pipeline:

TaskFlow API
Traditional syntax
import hashlib
import json

from airflow import DAG, AirflowException
from airflow.models import Variable
from airflow.models.baseoperator import chain
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import PythonOperator
from airflow.utils.dates import datetime
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.amazon.aws.transfers.local_to_s3 import (
    LocalFilesystemToS3Operator,
)
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.sql import SQLCheckOperator
from airflow.utils.task_group import TaskGroup


# The file(s) to upload shouldn't be hardcoded in a production setting,
# this is just for demo purposes.
CSV_FILE_NAME = "forestfires.csv"
CSV_FILE_PATH = f"include/sample_data/forestfire_data/{CSV_FILE_NAME}"


with DAG(
    "simple_redshift_3",
    start_date=datetime(2021, 7, 7),
    description="""A sample Airflow DAG to load data from csv files to S3
                 and then Redshift, with data integrity and quality checks.""",
    schedule=None,
    template_searchpath="/usr/local/airflow/include/sql/redshift_examples/",
    catchup=False,
):
    """
    Before running the DAG, set the following in an Airflow
    or Environment Variable:
    - key: aws_configs
    - value: { "s3_bucket": [bucket_name], "s3_key_prefix": [key_prefix],
             "redshift_table": [table_name]}
    Fully replacing [bucket_name], [key_prefix], and [table_name].
    """

    upload_file = LocalFilesystemToS3Operator(
        task_id="upload_to_s3",
        filename=CSV_FILE_PATH,
        dest_key="{{ var.json.aws_configs.s3_key_prefix }}/" + CSV_FILE_PATH,
        dest_bucket="{{ var.json.aws_configs.s3_bucket }}",
        aws_conn_id="aws_default",
        replace=True,
    )

    def validate_etag_function():
        """
        #### Validation function
        Check the destination ETag against the local MD5 hash to ensure
        the file was uploaded without errors.
        """
        s3 = S3Hook()
        aws_configs = Variable.get("aws_configs", deserialize_json=True)
        obj = s3.get_key(
            key=f"{aws_configs.get('s3_key_prefix')}/{CSV_FILE_PATH}",
            bucket_name=aws_configs.get("s3_bucket"),
        )
        obj_etag = obj.e_tag.strip('"')
        # Change `CSV_FILE_PATH` to `CSV_CORRUPT_FILE_PATH` for the "sad path".
        file_hash = hashlib.md5(open(CSV_FILE_PATH).read().encode("utf-8")).hexdigest()
        if obj_etag != file_hash:
            raise AirflowException(
                """Upload Error: Object ETag in S3 did not match
                hash of local file."""
            )

    validate_file = PythonOperator(
        task_id="validate_etag", python_callable=validate_etag_function
    )

    # --- Create Redshift Table --- #
    create_redshift_table = PostgresOperator(
        task_id="create_table",
        sql="create_redshift_forestfire_table.sql",
        postgres_conn_id="redshift_default",
    )

    # --- Second load task --- #
    load_to_redshift = S3ToRedshiftOperator(
        task_id="load_to_redshift",
        s3_bucket="{{ var.json.aws_configs.s3_bucket }}",
        s3_key="{{ var.json.aws_configs.s3_key_prefix }}" + f"/{CSV_FILE_PATH}",
        schema="PUBLIC",
        table="{{ var.json.aws_configs.redshift_table }}",
        copy_options=["csv"],
    )

    # --- Redshift row validation task --- #
    validate_redshift = SQLCheckOperator(
        task_id="validate_redshift",
        conn_id="redshift_default",
        sql="validate_redshift_forestfire_load.sql",
        params={"filename": CSV_FILE_NAME},
    )

    # --- Row-level data quality check --- #
    with open("include/validation/forestfire_validation.json") as ffv:
        with TaskGroup(group_id="row_quality_checks") as quality_check_group:
            ffv_json = json.load(ffv)
            for id, values in ffv_json.items():
                values["id"] = id
                SQLCheckOperator(
                    task_id=f"forestfire_row_quality_check_{id}",
                    conn_id="redshift_default",
                    sql="row_quality_redshift_forestfire_check.sql",
                    params=values,
                )

    # --- Drop Redshift table --- #
    drop_redshift_table = PostgresOperator(
        task_id="drop_table",
        sql="drop_redshift_forestfire_table.sql",
        postgres_conn_id="redshift_default",
    )

    begin = EmptyOperator(task_id="begin")
    end = EmptyOperator(task_id="end")

    # --- Define task dependencies --- #
    chain(
        begin,
        upload_file,
        validate_file,
        create_redshift_table,
        load_to_redshift,
        validate_redshift,
        quality_check_group,
        drop_redshift_table,
        end,
    )
