---
title: "ETL With Apache Airflow"
date: '2022-06-29'
categories:
  - miscellaneous
tags: 
- airflow
- apache
- etl
- api
classes: 
- wide
- text-justify
---

# Outline of the Project

I've slowly been expanding from data science into data engineering, and I thought building a full-stack pipeline would be a demonstration of what I can do. Personally, I also find it useful to document the process, in case I need to refer to it at a later date, or if someone else ends up doing a similar project.

The goal is to do all the ETL with airflow and python. Then, using the postgres database we'be created, query the relavent data and use machine learning to recommend songs based on my most recent listening patterns (using the recomendation algorithm I trained on Kaggle last year).

# Step 1. Install Docker with an Apache Airflow Image
For reference, I'm installing this on MacOS. First, [install the proper version of Docker Desktop](https://docs.docker.com/desktop/mac/install/) - this will include Docker CLI and Docker Compose.

Now we download the docker-compose yaml file via cURL:

```
curl -L -LfO 'https://airflow.apache.org/docs/stable/docker-compose.yaml'
 ```

Note that the `-L` parameter stands for `--location`, allowing curl to follow any redictions. In order to make the postgres database accessible outside of the container (let's say we wanted to explore the database using DBeaver, which is installed on our local machine), we need to add these two lines of code under the postgres section.

```
services:
  postgres:
    ...
    ...
    ports:
      - 5432:5432
```

I also had to add the port to my hosts to tell it to look for the localhost.

```
> nano ~/etc/hosts
# now add this line
5432 127.0.0.1
```

Next we add some relavent directories

```
 mkdir ./dags ./plugins ./logs
 ```
 and add the host user id and and sett the group id set to 0. Otherwise, the files created in dags, logs and plugins will be created with root user.
 ```
 echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_GID=0" > .env
 ```

 If you go into the .env file, it should look like this now:
 ```
AIRFLOW_UID=501
AIRFLOW_GID=0
 ```

Now, we use Docker Compose. The following command creates and starts the aiflow-init container. This container triggers the 6 required containers to fire up (airflow-triggers, airflow-worker, airflow-scheduler, airflow-webserver, postgres, and redis).
```
docker-compose up airflow-init
```

Now, go to [http://localhost:8080](http://localhost:8080) and login with airflow as the username and password (the defaults in the image). You should see lots of example dags that are included.

# Part 2: Automating the OAuth access token refresh
(Done, just writing up how I did it)

<!-- ## Miscellaneous notes for later
to do api calls
docker ps
curl -X GET --user "airflow:airflow" "http://localhost:8080/api/v1/dags"
```

To enter the bash of the python installation within docker being used for the dags
```
docker exec -ti d0a7f75e6335 bash
```
This might be useful to `pip install` something -->
<!-- 




generate an access. token automatically by gettin ga refresh token and using that to regenerate one. -->
<!-- curl -d client_id=0482e7425dbe4e529ccb26fa530d7261 -d client_secret=25bac7257e1f4843be265f1c09c96492 -d grant_type=authorization_code -d code=AQCYTA3vyvs7OcNEL9EGlI5SQ9CYrS6xki_gX21_4ow10Jwan7iuTFiVyBNq4Irpg8JwspE4qtX1p11DxXg7uZ2-w3PmK6UCEgoLO2cblAybcWuGZEmNuFQZ8507baP1iki2Jdi-w4zlu_ZYd-rI82lMF5817bchIIPWn3YX-k-qy59bXoUFf5T1Ow1-VGD1uLHtLhXNOAM -d redirect_uri=https://jeremythaller.com  https://accounts.spotify.com/api/token -->

<!-- https://benwiz.com/blog/create-spotify-refresh-token/ -->

<!-- _client_id = Variable.get("SPOTIFY_CLIENT_ID") -->
<!-- #     _client_secret = Variable.get("SPOTIFY_CLIENT_SECRET")
#     auth_url = 'https://accounts.spotify.com/api/token'
#     # client-credentials  'authorization-code', user-read-recently-played
#     auth_data = {'grant_type': 'authorization-code',
#                 'client_id': _client_id,
#                 'client_secret': _client_secret,
#                 'response_type': 'code'
#                 }
#     auth_response = requests.post(auth_url, data=auth_data)
#     access_token = auth_response.json().get('access_token') -->
<!-- https://github.com/EuanMorgan/SpotifyDiscoverWeeklyRescuer/blob/master/secrets.py -->
