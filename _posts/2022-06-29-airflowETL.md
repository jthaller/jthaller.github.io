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
Spotify now requires OAuth to gain API access to personal data, such as your recently played songs. The tricky thing with OAuth, is that it requires manual approval on a website to grant permissions. This is very difficult to do through airflow, because it's not intended to be run with interactive scripts. I thought about using selenium to interact with the webpage and grant permission, but I realized there's a much easier solution. Permsissions access only needs to be manually approved the first time. After doing this, Spotify gives a `refresh token` that can be used to gain an up-to-date `access token` upon request, without having to manually grant permissions through the web interface again.

To manually get the refresh token the first time, I ended up just doing it with `cURL` in the terminal, but you could do it with the `spotipy` package pretty easily if you want. 

```python
import spotipy
from spotipy.oauth2 import SpotifyOAuth
from spotipy.oauth2 import SpotifyPKCE

CLIENT_ID = ''
CLIENT_SECRET = ''
REDIRECT_URI = '' # example I used "https://jeremythaller.com"

sp = spotipy.Spotify(auth_manager=SpotifyPKCE(client_id=CLIENT_ID,
                                                client_secret=CLIENT_SECRET,
                                                redirect_uri=REDIRECT_URI,
                                                scope=["user-read-recently-played",
                                                        "user-library-read",
                                                        "playlist-read-private"
                                                      ]
                                                )
                    )
results = sp.current_user_saved_tracks()
for idx, item in enumerate(results['items']):
    track = item['track']
    print(idx, track['artists'][0]['name'], " – ", track['name'])
```

To do in the the CLI, after creating an app with my spotify developers account, and assigning a URI to my app (important, don't forget to do this). To get the `$CODE` part for the terminal command, you can just visit this url in your browser and grant permissions. For `$SCOPE` I used `user-read-recently-played`. I'm not sure if you can pass it a list with multiple scopes. And for my `REDIRECT_URI` I just set it up to be https://jeremythaller.com.

```bash
https://accounts.spotify.com/authorize?response_type=code&client_id=$CLIENT_ID&scope=$SCOPE&redirect_uri=$REDIRECT_URI
```

This will open you browser and you'll grant it permissions. It'll forward you to your redirect_uri with an addition to the URL called `code=$CODE`. Add that code and a few more parts and run this terminal command
```bash
curl -d client_id=$CLIENT_ID -d client_secret=$CLIENT_SECRET -d grant_type=authorization_code -d code=$CODE -d redirect_uri=$REDIRECT_URI  https://accounts.spotify.com/api/token
```
This will a JSON, and you can store your `client_id`, `client_secret`, and `refresh token` as variables within airflow. You also need to store the [64-bit encoded](https://www.base64encode.org/) `client_id`:`client_secret$` value too. I saved it in Airflow as a variable called `BASE_64_ID:SECRET`. Now, you can retreive these to refresh your token whenever you need to run an API call. Here's a function I wrote to grab you a refreshed token after having stored those three values in Airflow: 

```python
from airflow.models import Variable

def refresh_api_token():
        refresh_token = Variable.get("REFRESH_TOKEN")
        base_64_id_secret = Variable.get("BASE_64_ID:SECRET")

        query = "https://accounts.spotify.com/api/token"

        response = requests.post(query,
                                 data={"grant_type": "refresh_token",
                                       "refresh_token": refresh_token},
                                 headers={"Authorization": "Basic " + base_64_id_secret})

        response_json = response.json()
        print(response_json)
        return response_json["access_token"]

token = refresh_api_token()
```





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




<!-- https://benwiz.com/blog/create-spotify-refresh-token/ -->

<!-- https://github.com/EuanMorgan/SpotifyDiscoverWeeklyRescuer/blob/master/secrets.py -->
