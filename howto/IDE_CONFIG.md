# IDE Config

## Set Python SDK for your project

1. Check how to create custom virtual environment
   [here](https://umusic.atlassian.net/wiki/spaces/DATA/pages/110330032/Use+custom+Python+virtual+environment+direnv#Virtualenv)

1. Execute the following command to get the `requirements.txt` file from the local composer dev environment

    ```shell
    cd <ABSOLUTE/PATH/TO/data-platform-germany-workflows>
    docker ps
    docker exec -it <CONTAINER_ID> /bin/bash -c "pip list --format=freeze" > requirements.txt
    ```

1. Install pip requirements from requirements.txt file
   ```
   source ./venv/bin/activate
   pip install -r requirements.txt
   ```

1. in IntelliJ IDEA
   ```
   File > Project Structure > Edit Configuration > Left Sidebar SDKs > + > Add Python SDK > Virtual Enviornment > Existing
   environment > ... > Locate venv/bin/python > OK
   ```
   ```
   In `File > Project Structure > Edit Configuration > Left Sidebar Project > SDK: > Specify newly created Python SDK
    ```
1. in Visual Studio Code
   ```json
   {
     "python.defaultInterpreterPath" : "${workspaceFolder}/venv"
   }
   ```
   Now your VS Code terminal should automatically open in virtual environment.

## Add Python SDK sources to PYTHONPATH

Once you clone your repository that hosts airflow dags make sure to add

- `dags/`,
- `plugins/`

to PYTHONPATH.

1. in IntelliJ IDEA
   ![](img/intellij-idea-mark-as-source.png)
1. in Visual Studio Code

   Create `.vscode/settings.json`:
    ```json
    {
        "python.analysis.extraPaths": [
            "./plugins",
            "./dags"
        ]
    }
    ```
