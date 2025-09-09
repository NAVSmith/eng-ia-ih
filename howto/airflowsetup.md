# README

This repository is considered as the "Data Platform Germany", it is home for Airflow dags for UMG Germany.

## Getting Started

You need to have a Mac machine. The business reason is that [Airflow does not run in Window yet](https://github.com/apache/airflow/issues/10388).

You need to have the following roles attached to your Google Profile. If you want to learn more about the google groups setup, there is [documentation](https://umusic.atlassian.net/l/c/1rt2fawi).

- For Data Analysts, you have to be in this group: `umg.res.gcp-de-data-platform-analysts-user.gs`.
- If you are a Data Platform Engineer, the following group is for you: `umg.res.gcp-de-data-platform-engineer-owner.gs`.

Please send email to AppServices@umusic.com with CC DeBerTeamDataEng@umusic.com The setup of AD Group should be the same as
Google Cloud Composer SVC is using, to avoid issues like "it works on my machine."

# Set-up guide

### Git

- [Check here how to set up your git configuration](./GIT_CONFIG.md)

### Python & Pip
Our Airflow instances run in Python 3.11.8. Therefore, we recommend to install this version of Python as well.

- we support python installations and python version management with the help of `pyenv`
- prerequisites to install python with the help of `pyenv` are [Homebrew](https://docs.brew.sh/Installation) and [Xcode Command Line Tools](https://mac.install.guide/commandlinetools/4.html)
- before you begin the installation, please make yourself familiar with the concepts of `PATH` and `shims`: https://github.com/pyenv/pyenv#how-it-works
- install `pyenv` using homebrew: https://github.com/pyenv/pyenv#installation
- restart your shell and verify your installation by running `pyenv --version` in your shell
  - if you receive the pyenv version as a result you can go on with the next step
  - if not, please set up your shell environment for pyenv and try again: https://github.com/pyenv/pyenv#set-up-your-shell-environment-for-pyenv
- install python build dependencies using homebrew: https://github.com/pyenv/pyenv/wiki#suggested-build-environment
- install `Python 3.11.8`: https://github.com/pyenv/pyenv#install-additional-python-versions
- activate `Python 3.11.8` as needed: https://github.com/pyenv/pyenv#switch-between-python-versions
- to check what Python you are operating: `pyenv which python`
- if you like to know more about pyenv, we recommend to read this: https://realpython.com/intro-to-pyenv

We recommend to have a virtual environment for each repository to avoid package dependency issues. Here is how to set up virtual environments:

- https://umusic.atlassian.net/wiki/spaces/DATA/pages/110330032/Use+custom+Python+virtual+environment+direnv

### Google Cloud SDK
- [download latest version](https://cloud.google.com/sdk/docs/install)

### Airflow
- You can install a local Airflow instance based on Docker following [this guide](AIRFLOW_CONFIGURATION.md).
- To check for syntax errors while coding, we recommend installing Airflow in your virtual environment. ``pip install -r requirements.txt``

### Code Editor (IDE)
- Install **IDE** or code editor (e.g. Visual Studio Code, IntelliJ IDEA) following this [guide](IDE_CONFIG.md)

## Development

**Please** go through our [contribution guidelines](CONTRIBUTION_GUIDES.md) before coding. You can check the [analyst guide](ANALYST_GUIDE.md) for hands-on examples.

We enforce a python coding style which we check using pre-commit and CI. So you need to install pre-commit as follows.

```shell
brew install pre-commit
cd data-platform-germany-workflows
pre-commit install
```

Using pre-commit in order to format the python code:

```shell
pre-commit
```

Using pre-commit in order to format the SQL code:

```shell
 pre-commit run --files=<sql_file_path>  sqlfluff-fix
```

A successful commit should look like this
```shell
git commit -m "Update start date"
black....................................................................Passed
trim trailing whitespace.................................................Passed
check yaml...........................................(no files to check)Skipped
fix end of files.........................................................Passed
debug statements (python)................................................Passed
fix python encoding pragma...............................................Passed
flake8...................................................................Passed
[fix-finance-youtube bbc5172] Update start date
1 file changed, 1 insertion(+), 1 deletion(-)
```

You can verify your Google cloud setup as follows:
```shell
gcloud info
```
Look for the section "Current Properties"
```shell
$ gcloud config list
...

[core]
account = first.last@umusic.com
disable_usage_reporting = False
project = umg-de-restricted-nonprod

Your active configuration is: [default]
```

If by some reason is different, please reach out to the Slack support channel.

## How to deal with environments

You can write the BigQuery project explicitly neither in SQL nor in operators. So we use an environment variable instead.

### BigQuery Project

An example,

```sql
INSERT INTO umg-de.mydataset.mytable
SELECT col1
FROM umg-de.mydataset.sourcetable
```

This code would only work in `umg-de` but not the `umg-de-restricted-nonprod` project and break any test in staging. The
solution is to use the environment variable `BQ_PROJECT`.

In the Python example, we refer to the environment variable `BQ_PROJECT` and get the value
using `os.environ.get("AIRFLOW_VAR_BQ_PROJECT")` function.

```python
import os

# This represents the Composer environment (dev, staging, prod)
ENV = os.environ.get("AIRFLOW_VAR_ENV")

BQ_OUTPUT_PROJECT = os.environ.get("AIRFLOW_VAR_BQ_PROJECT")

bq_insert_revenues = BigQueryInsertJobOperator(
    task_id="bq_insert_revenues",
    gcp_conn_id="google_cloud_default",
    configuration={
        "query": {
            # This means that the SQL code is templated
            "query": "{% include './sql/bq_insert_revenues.sql' %}",
            "destinationTable": {
                "projectId": BQ_OUTPUT_PROJECT,
                "datasetId": "data_revenues",
                # Partitioning decorator $YYYYMMDD sets the partition as a destination instead of the whole table (incremental)
                "tableId": "fin_revenues${{ds_nodash}}",
            },
            time_partitioning={
                "field": "partition_date",
                "type": "DAY"
            },
            # It is required to have a table created upfront
            "createDisposition": "CREATE_NEVER",
            # TRUNCATE table (or partition) and then WRITE (makes task idempotent)
            "writeDisposition": "WRITE_TRUNCATE",
            "allowLargeResults": True,
            "useLegacySql": False,
        }
    },
)
```

In the SQL example, `{{var.value.BQ_PROJECT}}` allows us to get the variable directly.

```sql
SELECT
  a,
  b,
  c,
  DATE("{{ds}}") AS LOGICAL_DATE,
  CURRENT_TIMESTAMP() AS LOAD_TIME
FROM
  `{{var.value.BQ_PROJECT}}.my_cool_datset.my_cool_table`
WHERE
  partition_date = DATE("{{ds}}")
```

### Secrets

Data Platform uses Google Cloud Secret Manager to securely manage secrets (passwords, API tokens, credentials).
Airflow uses Secret Manager as Auth Backend for storing Airflow Connections and Variables.
Local Airflow environment shares secrets with non-prod Composer Airflow.

Variables:

| Project | Environment             | Prefix                                        |
|---------|-------------------------|-----------------------------------------------|
| umg-de  | umg-de-prod-composer    | airflow-variables-<CONNECTION_NAME>           |
| umg-de  | umg-de-nonprod-composer | airflow-variables-nonprod-<CONNECTION_NAME>   |
| umg-de  | composer-local-dev      | airflow-variables-nonprod-<CONNECTION_NAME>   |

Connections:

| Project | Environment             | Prefix                                        |
|---------|-------------------------|-----------------------------------------------|
| umg-de  | umg-de-prod-composer    | airflow-connections-<VARIABLE_NAME>           |
| umg-de  | umg-de-nonprod-composer | airflow-connections-nonprod-<VARIABLE_NAME>   |
| umg-de  | composer-local-dev      | airflow-connections-nonprod-<VARIABLE_NAME>   |

:warning: Whenever you develop a new DAG that requires a new secret, please request the creation of the secret with Data Engineering Team.

e.g. Please create a new secret `ftp_spotify_credentials`:
- `airflow-connections-ftp_spotify_credentials` (for production)
- `airflow-connections-nonprod-ftp_spotify_credentials` (for staging, local).

## Frequently Asked Questions

### 1. **I want to test a DAG locally. What can I do?**

There are multiple ways to test a DAG.

  a) You can test each task of the DAG. You need to provide an execution date at the end.
  ```shell
  airflow tasks test mydag my_task 2022-02-01
  ```
  b) You can check the rendering of the task:
  ```shell
  airflow tasks render mydag my_task 2022-02-01
  ```

### 2. **I need a library in Google Cloud Composer. What can I do?**

Get familiar with Docker and Airflow's operator KubernetesPodOperator. In case a library is used in multiple DAGs, owner can
install the library at cluster level.

### 2. **My personal or/and Cloud Composer Service Account need access to a specific dataset?**
  ```
  SUBJECT: Request for <personal/service-account> access in BigQuery for <dataset/table> <project:dataset.table>.
  TO: AppServices@umusic.com
  CC: DataGovernance@umusic.com, DeBerTeamDataEng@umusic.com

  Dear AppServices,

  Please assign this ticket to UMG Global Data Governance.

  ----

  Hi Data Governance,

  For my data pipeline, which resolves <<here write your business reason>>,

  I need the following policy.

  Principals:
  - first.last@umusic.com (remark for steward: in case many, check whether group applies)
  - (EU GitHub Composer) svc-umg-de-nonprod-eu-composer@umg-de-restricted-nonprod.iam.gserviceaccount.com
  - (EU GitHub Composer) svc-umg-de-prod-eu-composer@umg-de-restricted-prod.iam.gserviceaccount.com

  Role:
  a) organizations/490778785469/roles/UMGBigQueryReadOnly (READ)
  b) organizations/490778785469/roles/UMG_BQ_READ_ONLY_W_EXPORT (READ, EXPORT)
  c) organizations/490778785469/roles/UMG_BQ_READ_WRITE_UPDATE (READ, EXPORT, WRITE)
  d) organizations/490778785469/roles/UMG_BQ_READ_WRITE_UPDATE_DELETE (READ, EXPORT, WRITE, DELETE)

  Resource
  - dataset my_cool_project:my_cool_dataset
  - table my_cool_project:my_cool_dataset.my_cool_table

  Could you please review, approve and delegate access implementation?

  Kind regards,
  <Your Name>
  ```
