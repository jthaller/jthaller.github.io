---
title: "ETL With Apache Airflow"
date: '2022-06-29'
categories:
  - miscellaneous
  - data engineering
tags: 
- airflow
- apache
- etl
- api
classes: 
- wide
- text-justify
---

# Work in Progress - just jotting down some notes so I don't forget what I've done.

I've slowly been expanding from data science into data engineering, and I thought building a full-stack pipeline would be a demonstration of what I can do. Personally, I also find it useful to document the process, in case I need to refer to it at a later date, or if someone else ends up doing a similar project.

# Step 1. Install Docker with an Apache Airflow Image
I'm on a mac, so 

https://docs.docker.com/desktop/mac/install/ 

```
 curl -L -LfO 'https://airflow.apache.org/docs/stable/docker-compose.yaml'
 mkdir ./dags ./plugins ./logs
 echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0" > .env
docker-compose up airflow-init
 docker ps
curl -X GET --user "airflow:airflow" "http://localhost:8080/api/v1/dags"
```

To enter the bash of the python installation within docker being used for the dags
```
docker exec -ti d0a7f75e6335 bash
```
This might be useful to `pip install` something
