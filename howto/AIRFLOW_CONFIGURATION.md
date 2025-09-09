# Installing Airflow locally

Our goal is to install Airflow mirroring the installation of Google Cloud Composer v2.

For that, we use the offical GCP tool for running Composer locally - exactly corresponding to the Airflow configuration of the cloud version.

# 0. Requirements

Before continuing make sure you have:
- [Rancher Desktop](https://umusic.atlassian.net/wiki/spaces/DATA/pages/386039999/Docker) installed and it is running.
- [yq](https://mikefarah.gitbook.io/yq/v/v3.x/)

# 0.1 Define the composer version

```shell
mkdir -p ~/Projects/github/umg/
cd ~/Projects/github/umg/
git clone git@github.com:GoogleCloudPlatform/composer-local-dev.git
cd ./composer-local-dev
```

Here we export the parameters we need to set up:

`data-platform-germany-workflows` project: [Github](https://github.com/umg/data-platform-germany-workflows)
```shell
export COMPOSER_GIT="github"
export COMPOSER_REPO_NAME="data-platform-germany-workflows"
export COMPOSER_ENVIRONMENT_NAME="umg-de-nonprod-eu-composer-local"
export COMPOSER_ENVIRONMENT_PROJECT="umg-de-restricted-nonprod"
export COMPOSER_SOURCE_ENVIRONMENT="umg-de-nonprod-eu-composer"
export COMPOSER_LOCATION="europe-west1"
export AIRFLOW_REPO_URL="git@github.com:umg/data-platform-germany-workflows.git"
```

# 1. Install the Local Composer CLI tool

Set up and activate your virtual environment and install packages
```shell
# install python at OS level
pyenv install 3.11.8
# copy OS level python to local dir
pyenv local 3.11.8
virtualenv --python=python3.11.8 .venv
source .venv/bin/activate
pip install .
```

# 2. Configure GCP crendentials

```shell
# User personal credentials
gcloud auth login
# Machine personal credentials
gcloud auth application-default login --disable-quota-project
# Set the right Google Cloud Project
gcloud config set project $COMPOSER_ENVIRONMENT_PROJECT
```

# 3. Install Composer Staging instance locally

If you don't currently have the Airflow Repository in the path ``
please clone it here and use this folder from now on or replace ever reference to this path with your repository location.
Make sure you commited all your local changes before changing cloning the repository.
```shell
git clone $AIRFLOW_REPO_URL ~/Projects/$COMPOSER_GIT/umg/$COMPOSER_REPO_NAME
```

```shell
cd ~/Projects/github/umg/composer-local-dev
composer-dev create $COMPOSER_ENVIRONMENT_NAME \
    --from-source-environment $COMPOSER_SOURCE_ENVIRONMENT \
    --location $COMPOSER_LOCATION \
    --project $COMPOSER_ENVIRONMENT_PROJECT \
    --port 8081 \
    --dags-path ~/Projects/$COMPOSER_GIT/umg/$COMPOSER_REPO_NAME/dags \
    --database postgresql
```

**Note:** if you faced an error   ``` AttributeError: 'Credentials' object has no attribute 'universe_domain'** ```
Following the [issue 32](https://github.com/GoogleCloudPlatform/composer-local-dev/issues/32)
You might want to try modifying `"google-auth==1.30.*",` in file **composer-local-dev/setup.py** and bumping it up to `2.27.*` then try again

This will create a folder `./composer` inside the `../composer-local-dev` dir where setup for your local Composer instance (and all others) will be stored.

If you get the error: `Error while fetching server API version: request() got an unexpected keyword argument 'chunked'` try updating your docker pip version:
`python3 -m pip install docker --upgrade`

(AS OF MAY 2024-- still the case as of September 2024)
If you get the error:
```
ERROR: Docker not available or failed to start. Please ensure docker service is installed and running. Error: Error while fetching server API version: Not supported URL scheme http+docker
```
try downgrading your `requests` packtage for python from 2.32.* to 2.31.*:
while in the virtual environment run
`pip install requests==2.31.*`

# 4. Download enviroment variables from Google Composer to your local Composer instance

Run the following:
```shell
gcloud composer environments describe $COMPOSER_SOURCE_ENVIRONMENT \
  --project $COMPOSER_ENVIRONMENT_PROJECT \
  --location $COMPOSER_LOCATION \
    | yq .config.softwareConfig.envVariables \
    | sed 's/: /=/' \
    > ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env
```


# 5. Change / add environment variables for local development and to access Secrets Manager

Execute
```shell
sed -i "" "s/AIRFLOW_VAR_ENV=staging/AIRFLOW_VAR_ENV=dev/g" ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env
echo 'AIRFLOW_COMPOSER_DEV=true' >> ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env
echo 'AIRFLOW__CORE__DAGBAG_IMPORT_TIMEOUT=300' >> ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env

gcloud composer environments describe $COMPOSER_SOURCE_ENVIRONMENT \
  --project $COMPOSER_ENVIRONMENT_PROJECT \
  --location $COMPOSER_LOCATION \
  --format=json \
  | jq -r .config.softwareConfig.airflowConfigOverrides.\"secrets-backend\" \
  | tr -d " " \
  | xargs -0 printf AIRFLOW__SECRETS__BACKEND=%s \
  >> ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env

gcloud composer environments describe $COMPOSER_SOURCE_ENVIRONMENT \
  --project $COMPOSER_ENVIRONMENT_PROJECT \
  --location $COMPOSER_LOCATION \
  --format=json \
  | jq -r .config.softwareConfig.airflowConfigOverrides.\"secrets-backend_kwargs\" \
  | tr -d " " \
  | xargs -0 printf AIRFLOW__SECRETS__BACKEND_KWARGS=%s \
  >> ./composer/$COMPOSER_ENVIRONMENT_NAME/variables.env
```

**Note**
To verify the secrets you cat check the `variables.env` file using `cat ~/Projects/$COMPOSER_GIT/umg/composer-local-dev/composer/$COMPOSER_ENVIRONMENT_NAME/variables.env`
You should be able to see the 2 variables
```
AIRFLOW__SECRETS__BACKEND=airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend
AIRFLOW__SECRETS__BACKEND_KWARGS={"connections_prefix":"airflow-connections-nonprod","project_id":"umg-de-restricted-nonprod","sep":"-","variables_prefix":"airflow-variables-nonprod"}
```

# 6. Set plugins

```shell
rm -r ./composer/$COMPOSER_ENVIRONMENT_NAME/plugins
cp -r ~/Projects/$COMPOSER_GIT/umg/$COMPOSER_REPO_NAME/plugins ./composer/$COMPOSER_ENVIRONMENT_NAME
```

Note:
Currently, the `config.json` supports only specifications for dag folder path. Plugins are currently not supported. This is why dags folder copy can be skipped. This is clearly redundant step but it is necessary at the current stage.


# 7. Start and run local Composer

Start the composer instance:
```shell
composer-dev start $COMPOSER_ENVIRONMENT_NAME
```

Starting the composer instance for the first time will take some time.

Verify by going to the UI:
1. Open your web browser
2. Go to [localhost:8081](http://localhost:8081/home)
3. View if Airflow is running

If the UI is visible and there are no DAG import errors, the step has been successful.

If you receive `No response from gunicorn master within 120 seconds` change `dag_dir_list_interval` to 300
in `config.json`. You can access and change the file like so

```shell
nano ~/Projects/github/umg/composer/umg-de-nonprod-eu-composer-local/config.json
```

# 8. Verify access to Cloud Secret Manger

Shell in into the running docker container that is running composer locally and test acces:
```shell
docker ps
docker exec -it <CONTAINER_ID> /bin/bash

# for testing you can try to fetch the onboarding-dummy variable
airflow variables get onboarding-dummy

# to fetch any variable you can use
airflow variables get <variable-stored-in-secrets-manager>
```
If the command returns the variable, the step has been successful.

# 9. Improve performance with .airflowignore
To your dags folder add a `.airflowignore` file. It should look just like this:
```
dag_
```

This file excludes all dag folders and files that start with `dag_`. As this is our naming convention you'll see nothing
in Airflow when you start up. Once you want to test a dag locally you just rename the folder and/or the dag file and
it will show up in you local Airflow installation. This way composer-dev is acting much faster.

Once you're done with testing and you want to deploy make sure you revert the naming.
You can adjust the `.airflowignore` file as you need. It won't be commited to the repository.

More information about `.airflowignore` [here](https://airflow.apache.org/docs/apache-airflow/1.10.9/concepts.html#airflowignore).

# 10. (Advanced) Enable K8s access credentials

Note: https://github.com/GoogleCloudPlatform/composer-local-dev?tab=readme-ov-file#interaction-with-kubernetes-clusters

Before starting container, add the following environment variable to your `~/.zshrc` profile:
```shell
export KUBECONFIG=~/.kube/config
composer-dev start $COMPOSER_ENVIRONMENT_NAME

# verify kubeconfig file by executing `cat ~/.kube` within the container
```

Test the connection to a K8S cluster
```shell
docker ps
docker exec -it <CONTAINER_ID> /bin/bash
kubectl config get-contexts # -> should show existing contexts
kubectl config use-context <CLUSTER> # -> select cluster
kubectl get namespace # -> see namespaces in cluster
kubectl get pods -n <NAMESPACE> # -> should return list of pods
```

# 11. (Advanced) Enable Apache Beam Dataflow access credentials

When running Apache Beam with Dataflow Runner

a) without Airflow locally
  - your private Application Default Credentials (ADC) will be used.
  - make sure that `--service_account_name` or `BeamRunPythonPipelineOperator.service_account_name` is **not** specified when running locally.
  - check here for example: https://github.com/umg/data-platform-germany-playground-beam-dataflow/tree/main

b) with Airflow locally
  - Please specify the following (check [https://umusic.atlassian.net/wiki/spaces/DATA/pages/518357090/Dataflow+FAQ](https://umusic.atlassian.net/wiki/spaces/DATA/pages/518357090/Dataflow+FAQ#:~:text=How%20to%20run%20airflow%20pipeline%20that%20submits%20dataflow%20job%20locally%3F))
