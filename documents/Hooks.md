## Airflow hooks
A hook is an abstraction of a specific API that allows Airflow to interact with an external system. Hooks are built into many operators, but they can also be used directly in DAG code.

In this guide, you'll learn about using hooks in Airflow and when you should use them directly in DAG code. You'll also implement two different hooks in a DAG.


Assumed knowledge
To get the most out of this guide, you should have an understanding of:

Basic Airflow concepts. See Introduction to Apache Airflow.
Basic Python. See the Python Documentation.
Hook basics
Hooks wrap around APIs and provide methods to interact with different external systems. Hooks standardize how Airflow interacts with external systems and using them makes your DAG code cleaner, easier to read, and less prone to errors.

To use a hook, you typically only need a connection ID to connect with an external system. For more information about setting up connections, see Manage your connections in Apache Airflow.

All hooks inherit from the BaseHook class, which contains the logic to set up an external connection with a connection ID. On top of making the connection to an external system, individual hooks can contain additional methods to perform various actions within the external system. These methods might rely on different Python libraries for these interactions. For example, the S3Hook relies on the boto3 library to manage its Amazon S3 connection.

The S3Hook contains over 20 methods to interact with Amazon S3 buckets. The following are some of the methods that are included with S3Hook:

check_for_bucket: Checks if a bucket with a specific name exists.
list_prefixes: Lists prefixes in a bucket according to specified parameters.
list_keys: Lists keys in a bucket according to specified parameters.
load_file: Loads a local file to Amazon S3.
download_file: Downloads a file from the Amazon S3 location to the local file system.
When to use hooks
Since hooks are the building blocks of operators, their use in Airflow is often abstracted away from the DAG author. However, there are some cases when you should use hooks directly in a Python function in your DAG. The following are some general guidelines for using hooks in Airflow:

Hooks should always be used over manual API interaction to connect to external systems.
If you write a custom operator to interact with an external system, it should use a hook.
When an operator with built-in hooks exists for your specific use case, you should use the operator instead of manually setting up a hook.
If you regularly need to connect to an API and a hook is not available, write your own hook and share it with the community.
Example implementation
The following example shows how you can use the hooks (S3Hook and SlackHook) to retrieve values from files in an Amazon S3 bucket, run a check on them, post the result of the check on Slack, and then log the response of the Slack API.

For this use case, you'll use hooks directly in your Python functions because none of the existing Amazon S3 operators can read data from multiple files within an Amazon S3 bucket. Also, none of the existing Slack operators can return the response of a Slack API call, which you might want to log for monitoring purposes.

The source code for the hooks used in this example can be found in the following locations:

S3Hook source code
SlackHook source code
Prerequisites
Before running the example DAG, make sure you have the necessary Airflow providers installed. If you are using the Astro CLI, add the following packages to your requirements.txt file:

apache-airflow-providers-amazon
apache-airflow-providers-slack

Create the connections
In the Airflow UI, go to Admin > Connections and click the plus (+) icon to add a new connection.
In the Connection Id field, enter a unique name for the connection.
In the Connection Type list, select Amazon S3 as the connection type for the Amazon S3 bucket. If the Amazon S3 connection type isn't available, make sure you installed the provider correctly.
Enter your AWS access key ID in the Login field.
Enter your AWS secret access key in the Password field. To retrieve your AWS access key ID and AWS secret access key, see AWS Account and Access Keys.
Click Save.
Repeat steps 1 to 6 to create a new connection for Slack. Select Slack Webhook as the connection type and enter your Bot User OAuth Token in the Password field. To obtain the token, go to Features > OAuth & Permissions on api.slack.com/apps.
Run the example DAG
The following example DAG uses Airflow Decorators to define tasks and XCom to pass information between Amazon S3 and Slack. The name of the Amazon S3 bucket and the names of the files that the first task reads are stored as environment variables for security purposes.

The following example DAG completes the following steps:

A Python task with a manually implemented S3Hook reads three specific keys from Amazon S3 with the read_key method and then returns a dictionary with the file contents converted to integers.
A second Python task completes a simple sum check using the results from the first task.
The SlackHook call method posts the sum check results to a Slack channel and returns the response from the Slack API.
TaskFlow API
Traditional syntax
# importing necessary packages
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.slack.hooks.slack import SlackHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

# set bucket name and file names
S3BUCKET_NAME = "myhooktutorial"
S3_EXAMPLE_FILE_NAME_1 = "file1.txt"
S3_EXAMPLE_FILE_NAME_2 = "file2.txt"
S3_EXAMPLE_FILE_NAME_3 = "file3.txt"


# function to read 3 keys from your S3 bucket
def read_keys_from_s3_function():
    s3_hook = S3Hook(aws_conn_id="aws_conn")
    response_file_1 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_1, bucket_name=S3BUCKET_NAME
    )
    response_file_2 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_2, bucket_name=S3BUCKET_NAME
    )
    response_file_3 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_3, bucket_name=S3BUCKET_NAME
    )

    response = {
        "num1": int(response_file_1),
        "num2": int(response_file_2),
        "num3": int(response_file_3),
    }

    return response


# function running a check on the data retrieved from your S3 bucket
def run_sum_check_function(response):
    if response["num1"] + response["num2"] == response["num3"]:
        return (True, response["num3"])
    return (False, response["num3"])


# function posting to slack depending on the outcome of the above check
# and returning the server response
def post_to_slack_function(sum_check_result):
    slack_hook = SlackHook(slack_conn_id="hook_tutorial_slack_conn")

    if sum_check_result[0] is True:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""All is well in your bucket!
                        Correct sum: {sum_check_result[1]}!""",
            },
        )
    else:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""A test on your bucket contents failed!
                        Target sum not reached: {sum_check_result[1]}""",
            },
        )

    # return the response of the API call (for logging or use downstream)
    return server_response


# implementing the DAG
with DAG(
    dag_id="hook_tutorial",
    start_date=datetime(2022, 5, 20),
    schedule="@daily",
    catchup=False,
    # Render templates using Jinja NativeEnvironment
    render_template_as_native_obj=True,
):
    read_keys_form_s3 = PythonOperator(
        task_id="read_keys_form_s3", python_callable=read_keys_from_s3_function
    )

    run_sum_check = PythonOperator(
        task_id="run_sum_check",
        python_callable=run_sum_check_function,
        op_kwargs={
            "response": "{{ ti.xcom_pull(task_ids='read_keys_form_s3', \
                key='return_value') }}"
        },
    )

    post_to_slack = PythonOperator(
        task_id="post_to_slack",
        python_callable=post_to_slack_function,
        op_kwargs={
            "sum_check_result": "{{ ti.xcom_pull(task_ids='run_sum_check', \
                key='return_value') }}"
        },
    )

    # the dependencies are automatically set by XCom
    read_keys_form_s3 >> run_sum_check >> post_to_slack
