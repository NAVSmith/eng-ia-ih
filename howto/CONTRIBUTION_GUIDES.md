# Contribution Guidelines.


[[_TOC_]]


## SQL Style

Today, there is no SQL Code checker available in the market. Therefore the following patterns must be followed by analysts.


### Top to bottom writing with CTEs

Analysts and engineers should structure the code with CTEs (common table expression), which reads from top to bottom, novice-versa. Where performance permits, CTEs should perform a single, logical unit of work.
The SQL statement within the CTE should be intended with four spaces, whereas the query itself should be intended with two spaces.

```sql
 -- Good
WITH sales AS (
    SELECT
      week,
      SUM(sales) AS sales_per_week
    FROM sales_table
    WHERE week >= TO_DATE(‘2021-09-01’,’YYYY-MM-DD’)
    GROUP BY 1
), costs_of_sales AS (
    SELECT
      week,
      SUM(costs) as cost_per_week
    FROM costs
    WHERE week >= TO_DATE(‘2021-09-01’,’YYYY-MM-DD’)
    GROUP BY 1
)
SELECT

……
------------------------------------

-- Bad

SELECT
    a.week
    a.sales_per_week,
    b.cost_per_week
FROM (
   SELECT
     week,sales_per_week
     FROM (
    SELECT
      week,
              SUM(sales) AS sales_per_week
            FROM sales
    GROUP BY 1)
  WHERE week >= TO_DATE(‘2021-09-01’,’YYYY-MM-DD’)
) sales
LEFT JOIN …..
```

### Field naming and naming convention

The fields should be snake-cased and lowercase.
```sql
-- Good

SELECT
  dvccretts AS device_created_timestamp
FROM ….


-- Bad

SELECT
  dvccretts AS DeviceCreatedTimestamp
FROM ….
```


The fields should not have ambiguous names.

```sql
-- Good
SELECT
  id AS account_id,
  type AS account_type,
  date AS acount_creation_date
FROM accounts




--  Bad
SELECT
  id,
  type,
  date
FROM accounts
```


Boolean field names should start with has_, is_, or does_

```sql
  -- Good
  SELECT
    deleted AS is_deleted,
    sla     AS has_sla
  FROM table

  -- Bad
  SELECT
    deleted,
    sla
  FROM table
```




## Airflow Convention

### DAG folder naming


To make use of `.airflowignore` and the corresponding performance improvement for local development, all DAG folders
have the prefix `dag_`. More details [here](AIRFLOW_CONFIGURATION.md).


### Keep tasks atomic and idempotent.


Airflow tasks should perform one task which should run independently and produce the same output.
For example, a pipeline that deals with Instagram data should have at least three tasks:
Extract the data from Instagram API
Load into the marketing data lake
Transform data for the marketing reporting layer.



### Use Airflow default variables.

Airflow has many default variables and macros, which is better than creating custom Python code:

https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html



### Treat your DAG file as a “config file”.


Your DAG should be as lean as possible. For example, you should keep python or SQL code in different files, and the DAG can import these as either files or libraries. In addition, there is a rich set of operators within Airflow that will free you to write “custom code”.




### Usage of intermediate storage



We highly recommend storing interim results in a file system or to a database, as part of making the DAGs idempotent. This approach also helps in reducing memory usage. For example, you could face the following problem:


```
BigQuery job failed. Final error was: {'reason': 'resourcesExceeded', 'message': 'Resources exceeded during query execution: The query could not be executed in the allotted memory. Peak usage: 129% of limit.\nTop memory consumer(s):\n  sort operations used for analytic OVER() clauses: 98%\n  other/unattributed: 2%\n'}.
```



Example.
We have to create a cube with metrics for marketing performance reporting. You could create a single SQL file to create the cube; however, that comes at memory costs.

Instead, we do the following.

- One Task with one SQL file which calculates the necessary dimensions before aggregation.
- One Task with one SQL creates a set of metrics given a timeline, for example, per release date (table t1).
- One Task with one SQL creates a set of metrics given a timeline, for instance, per calendar date (table t2).
- Finally, one Task which UNION ALL the two tables (t1 and t2)
