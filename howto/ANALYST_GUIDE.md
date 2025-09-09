# How to develop my first DAG

[[_TOC_]]


## First step: Check out a new branch
We first check out the `main` branch. We need to get the latest changes done. Next step, we create our new branch `my_test_example`

```shell
git checkout main
git pull
> Your branch is up to date with 'origin/main'.
git checkout -b my_test_example
```

The following message means that it is OK.
```shell
> Switched to a new branch 'test_branch_test'
```

If you get the following message, you need to resolve the conflicts. This [guide](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line) gives you a good overview.

```shell
git status
> On branch main
> You have unmerged paths.
> (fix conflicts and run "git commit")
> (use "git merge --abort" to abort the merge)
>
> Unmerged paths:
> (use "git add <file>..." to mark resolution)
>
> both modified:   yourfile.txt
> no changes added to commit (use "git add" and/or "git commit -a")
```


## Second Step: create yourdag.py


This file will be your main configuration file. The comments in the code mean the explanation for each line. However, when you start your DAG, please remove them.  You can check the sample code in [this folder](/docs/mydagtest)

```python
# -*- coding: utf-8 -*-
# It is necessary to add a comment below to explain the purpose of the DAG
"""
D2C from source system Faktura into D2C model
The stakeholder is the DEPARTMENT. Please contact PERSON AT DEPARTMENT or deputy.

- This DAG creates a TABLE
"""
# this will allow you to retrieve the environment variable in
import os
# This provides you to create the DAG
from umg.umg_dag_factory import UmgDAG
# This will enable you to have a start or end visually. This approach also allows you to run multiple tasks and get together to the next step.
from airflow.operators.empty import EmptyOperator

from airflow.providers.google.cloud.operators.bigquery import (
    BigQueryInsertJobOperator,  # This allows you to load the result of a query in a BigQuery Table
)

# This is needed to define some time configuration at the DAG level
from datetime import datetime, timedelta
# This allows you to get the slack alert notification in either dev or prod alert channels.
from slack.plugin_slack import task_fail_slack_alert

# this will get the value of the variable you defined locally.

bq_prj = os.environ.get("AIRFLOW_VAR_BQ_PROJECT")


# for getting secrets out of Google Secret Manager, you need to
# from airflow.models import Variable
# my_secret = Variable.get("name of secret in Google Secret Manager‚")

with (
    UmgDAG.airflow_dag(
        dag_id="test",  # this is the name of the DAG
        schedule="0 09 27-31 * * ",  # this is the scheduling in cron format.
        max_active_runs=1,  # unless you have special requirements, this should not change
        concurrency=1,  # OK if you can run multiple tasks.
        description=__doc__.split("\n\n")[0],  # keep this
        dagrun_timeout=timedelta(hours=22),  # keep this
        owner="airflow",
        start_date=datetime(2025, 1, 1),
        slack_on_failure=task_fail_slack_alert,
        catchup=True,  # Keep this is you want to start your DAG for a given date, i.e. 2022-03-01 and backfill all data. If not set to False
    ) as dag
):

   start_dag = DummyOperator(task_id="start_dag", dag=dag) # this is the start of the DAG
   end_dag = DummyOperator(task_id="end_dag", dag=dag) # this is the end of the DAG

   DATASET_NAME = "my_dataset_name"

# Only necessary if the dataset doesn't exist
   create_my_schema  = BigQueryCreateEmptyDatasetOperator(
       dag=dag, #keep this
       task_id="create-dataset", # this is the name of the task
       dataset_id=DATASET_NAME,
       exists_ok=True, # this does not fail if schema exists.
   )


   load_data_into_bq = BigQueryInsertJobOperator(
       task_id="load_data_into_bq",
       configuration={
           "query": {
               "query": "{% include './sql/test_table.sql' %}",  # your SQL code is in the folder sql
               "destinationTable": {
                   "projectId": bq_prj, # variable which reflects the project ID, always use variables!
                   "datasetId": DATASET_NAME,
                   "tableId": "my_test_table", #in case of using partitions. ${{ds}}
               },
               "createDisposition": "CREATE_NEVER",
               "writeDisposition": "WRITE_TRUNCATE",
               "allowLargeResults": True,
               "useLegacySql": False,
           }
       },
       dag=dag,
       gcp_conn_id="google_cloud_default",
   )

   start_dag >> create_my_schema >> load_data_into_bq >> end_dag  # this defines the order of tasks

```



## Advanced Concepts

### Capture changed rows.

Scenario: I have a table containing source data, and I want to compare if there is new data from the file I am loading each day.

Approach:
- The first step, we have a task that loads the file to a staging table.
- The second step, we compare the existing table with the new records table and load them into the existing table

```sql
WITH new_data AS (
   SELECT *
   FROM  `{{var.value.BQ_PROJECT}}.your_dataset.my_staging_table`
), existing_table AS (
   --load_date does not exist in the staging table
   SELECT *  EXCEPT (load_date)
   FROM `{{var.value.BQ_PROJECT}}.your_dataset.brv_kunde`
), get_new_data as (
   SELECT * FROM new_data
   EXCEPT DISTINCT
   SELECT * FROM existing_table
)
SELECT *,
CURRENT_TIMESTAMP AS load_date
FROM get_new_data
```

### Use window functions.

Scenario: I have a set of calculations based on the same table, but not sure how to make it simpler.

Approach: We use [window functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/analytic-function-concepts)

For example, calculating customer sales metrics

```sql

SELECT DISTINCT
   customer_id,
   CAST(SUM(total_order_items_ordered) OVER (PARTITION BY customer_id) AS INT64) AS items_count_ordered,
   CAST(COUNT(DISTINCT CASE WHEN is_refund IS FALSE THEN order_id ELSE NULL END) OVER (PARTITION BY customer_id) AS INT64) AS order_count,
   CAST(COUNT(DISTINCT CASE WHEN is_refund IS FALSE THEN invoice_nr ELSE NULL END) OVER (PARTITION BY customer_id) AS INT64) AS invoice_count,
   CAST(COUNT(DISTINCT CASE WHEN is_refund IS TRUE THEN invoice_nr ELSE NULL END) OVER (PARTITION BY customer_id) AS INT64) AS refund_count,
   CAST(COUNT(DISTINCT order_sku) OVER (PARTITION BY customer_id) AS INT64) AS distinct_product_count,
   CAST(SUM(CASE WHEN is_pre_order IS TRUE THEN total_order_items_ordered ELSE NULL END) OVER (PARTITION BY customer_id) AS INT64) AS preorder_items_count_ordered,
   CAST(SUM(total_order_items_invoiced) OVER (PARTITION BY customer_id) AS INT64) AS items_count_invoiced,
   CAST(SUM(CASE WHEN is_pre_order IS TRUE THEN total_order_items_invoiced ELSE NULL END) OVER (PARTITION BY customer_id) AS INT64) AS  preorder_items_count_invoiced,
   CAST(COUNT(DISTINCT shop_code) OVER (PARTITION BY customer_id) AS INT64) AS shop_count,
   EXTRACT(DATE FROM MAX(CASE WHEN is_refund is FALSE THEN order_date ELSE NULL END) OVER (PARTITION BY customer_id)) AS last_purchase,
   ROUND(SUM(total_amount_invoiced) OVER (PARTITION BY customer_id) ,2) AS total_amount_invoiced,
   ROUND(SUM(total_shipping_costs_invoiced) OVER (PARTITION BY customer_id), 2) AS total_shipping_costs_invoiced,
   ROUND(SUM(total_net_sale_amount_invoiced) OVER (PARTITION BY customer_id), 2) AS total_net_sale_amount_invoiced,
   ROUND(SUM(total_gross_sale_amount_invoiced) OVER (PARTITION BY customer_id),2) AS total_gross_sale_amount_invoiced
 FROM `{{var.value.BQ_PROJECT}}.data_d2c.orders_denormalized`
```
### Partitioning Tables

**What is Partition in Big Query?**

- This page provides an overview of partitioned tables in BigQuery
- A partitioned table is a special table that is divided into partitions, that make it easier to manage and query your data
- By dividing a large table into smaller partitions, you can improve query performance, and you can control costs by reducing the number of bytes read by a query

![partitioning_illustration.png](img/partitioning_illustration.png)

**BigQuery example for better understanding**

Following situation:

| _**table_name**_                | **_Storage_**  |
|---------------------------------|----------|
| questions/questions_partitioned | 18 GB    |

The table `questions` is not partitioned. The table `questions_partitioned` is partitioned on the field `creation_date`.

**Without partition:**

![img_1.png](img/bq_query_without_without_partitioning.png)


**With partition:**

![img.png](img/bq_query_with_partitioning.png)

#### SUMMARY about Partitioning

At the end there are the same query request, but with using partitioning in one table we
waste 10 times less performance like in the example. At the end it saves costs, due to unnecessary waste of performance.

#### Different Types of Partitioning

**time-unit partitioning:**

- based on TIMESTAMP, DATE, or DATETIME
- Using a time-unit column field BigQuery can write and read information to and from the partitioned tables based on the time unit
- BigQuery can apply different granularity on your time unit partition field, so you can have more flexibility over the partitions

Example:

| partition_date	 | Partition (yearly) |
|-----------------|:------------------:|
| 2016-12-25      |       	2016        |
| 2018-11-21      |       	2018        |
| 2018-04-08      |       	2018        |

**Ingestion-time partitioning:**

- Ingestion-time partitioning is using the time when BigQuery ingests the data to partition the table in multiple chunks.
- In a similar way to the time-unit column partitioning, you can choose between hourly, daily, monthly, and yearly granularity for the partitions
- In order to be able to execute the queries, BigQuery creates two pseudo columns called _PARTITIONTIME and _PARTITIONDATE, where it stores the ingestion time for each row truncated by your granularity of choice (e.g., monthly).

_Example (monthly granularity):_

| Ingestion time       |   	_PARTITIONTIME	   | _PARTITIONDATE       | Partition (monthly) |
|----------------------|:--------------------:|:-----------------------|:-------------------:|
| 2016-12-25 18:08:04  | 	2016-12-25 18:08:04 | 2016-12-25 00:00:00   |     	201612         |
| 2018-11-21 09:24:54  | 	2018-11-21 09:24:54 | 2018-11-21 00:00:00   |     	201811         |
| 2018-04-08 23:19:11  |   2018-04-08 23:19:11 | 2018-04-08 00:00:00   |      201804         |

**Integer-range partitioning:**

- BigQuery partitions a table based on an integer-type column
- In order to create an integer-range partition table, you need to provide four arguments:

1. The integer-type column name.
2. The starting value for range partitioning (this is inclusive, so whatever value you’re using will be included in the partitions).
3. The ending value for range partitioning (this is exclusive, so whatever value you’re using will not be included in the partitions).
4. The interval between the starting and ending values.


##### ATTENTION:
BigQuery does not support partitioning by multiple columns. Only one column can be used to partition a table.


#### Code example how to create as well as load data in a partitoned table

Example DAG: https://github.com/umg/data-platform-germany-workflows/tree/main/dags/dag_example_bq_table_partitioning

In the example DAG we incrementally load data from a raw table into partitioned tables based on **time unit partitioning** and
**ingestion time partitioning**. On top of those partitioned tables we do incremental aggregations and save those
aggregations to again partitioned tables.

As an argument for the parameter you have to describe the partitioning like this:
- _For ingestion time partitioned tables_
```python
time_partitioning = {"type": "DAY"}
```
By choosing the type you define the granularity of your partitioning.

- _For time unit partitioned tables_
```python
time_partitioning ={"type": "DAY", "field": "date"}
```
By choosing the type you define the granularity of your partitioning.
The field indicates the column you're basing the partitioning on.
