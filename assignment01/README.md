# Pre-requisuite:
1. python and docker installed
2. vscode
3. airflow provider with postgres


## Please perform following Tasks:
1. Run docker to deploy postgres database
2. Run airflow webserver UI.
3. copy the dag file to airflow dag location
4. create connections for postgres and user_api


## Running postgres instance in docker 
docker pull postgres

docker run --name postgresql -e POSTGRES_USER=myusername -e POSTGRES_PASSWORD=mypassword -p 5432:5432 -v /data:/var/lib/postgresql/data -d postgres

docker ps | grep postgresql 
