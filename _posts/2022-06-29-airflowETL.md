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

## Sections

1. [Docker Setup](#Docker)
2. [OAuth Access Token](#access_token)
3. [Machine Learning](#ML)
4. [Push Playlist](#push)


# Step 1. Install Docker with an Apache Airflow Image<a id="Docker"></a>
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

# Part 2: Automating the OAuth Access Token Refresh<a id="access_token"></a>
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
                                 headers={"Authorization": "Basic {}".format(base_64_id_secret)}
                                 )

        response_json = response.json()
        return response_json["access_token"]

token = refresh_api_token()
```

# Part 3: Machine Learning and Creating a Recommendation Algorithm<a id="ML"></a>

There are a few ways to go about making recommendations. One basic algorithm that is typically used is called something like user-user recommendations. Essentially, you create a venn diagram of the listening history of two users, trying to maximize the overlap. Then what remains (the outer-join) are songs that person A has listened to but person B hasn't (and vice-versa). These are the songs you recommend to the other person. In practice, you could make a song-vector (like a word-vector in nlp) of recent listening history, then use cosine similarity to find the most similar user-listening-history. Then, do the outer-join and use those songs for the recommendation.

I've created two recommendation algorithms based on song vectors. The first, uses Spotify's own song features, and loops through all the songs in the database to find the most similar song via the cosine distance. A basic implimentation of this algorithm can be found in my Kaggle notebook: [Spotify-EDA with PySpark and Clustering](https://www.kaggle.com/code/fusshandschuhe/spotify-eda-with-pyspark-and-clustering). This algorithm becomes very memory intensive, as cosine comparisons rely on one-hot-encoded variable; you end up with a very high dimensional vector. In terms of computational complexity, it's not as bad. The comparison is O(n) time complexity, and is very easy to optimize via MapReduce. A MapReduce job partitions the input data-set into independent chunks, which are then processed by the map tasks in parallel. The framework sorts the outputs of the maps, which are then input to the reduce tasks.

I'm interested in incorporating NLP as part of the recommendation algorithm. Instead of considering song properties, consider playlists. Spotify has made publicly available a [dataset containing one million spotify playlists](https://research.atspotify.com/2020/09/the-million-playlist-dataset-remastered/). Using these playlists, we can create song embeddings and recommend a song based on what predicted song would come next in a playlist. The NLP analog is predicting which word comes next in a sentence using Word2Vec. This [Kaggle Notebok](https://www.kaggle.com/code/ashwinik/songs-recommendation-using-word2vec-on-playlist) does something very similar, and you can download their embeddings to use with my python scripts. At some point, I created my own embeddings and played around with skip-grams (predict missing song between two songs in a playlist) versus continuous bag of words (predict the next song in a playlist) as well as differnet window sizes, but I seem to have lost this notebook when I upgraded laptops.

Using the embeddings, it very easy to find similar songs. Gensim's word2vec package literally has a `most_similar()` function to give you the most similar songs in a semantically similar context. I used song_ids to simplify the embedding process (since they are unique), however, this meant I had to do a lot of dictionary reversing to extract the titles.

```python    
def similar_songs(self, song:tuple, n:int) -> list:
        """ Gets the songname from user and return the n similar songs """

        song_name = song[0]
        song_id = song[1]
        print ("Searching for songs similar to :", song_name)
        try:
            similar = self.model.most_similar(song_id, topn=n)
        except KeyError:
            print(f"couldn't find {song_name}, {song_id} for some reason")
            return []

        print ("Similar songs are as follow:")
        for id, _ in similar:
            print (self.reverse_titles_dict[id])
        return [self.reverse_titles_dict[id] for id, _ in similar]
```
```python
def create_recommendations(self) -> list:
        """
        No inputs. Get recent songs. Filter to just ones we have embeddings for.
        Get the 2 most similar songs for each. Return the URIs as a list of strings.
        """

        songs_df = self.get_recent_songs()

        # match the listening history to song embeddings
        matched_songs_ids = []
        for song in songs_df.song_name.values:
            try: # try to match and see if we have an embedding for the
                matched_songs_ids.append((song, self.song_title_dict[song.lower()]))
                print(song, '--', self.song_title_dict[song.lower()])
            except KeyError:
                print(f"{song} -- not found")
        
        print("Found {}/{} songs".format(len(matched_songs_ids), songs_df.shape[0]))

        similar_songs = [self.similar_songs(song=song_tuple, n=2) for song_tuple in matched_songs_ids]
        similar_songs =  [y for x in similar_songs for y in x]
        print(similar_songs)
        uris = [self.get_uri(song) for song in similar_songs]
        print(uris)
        return uris
```

# Part 4: Push this ML-recommended Playlist to Spotify Account<a id="push"></a>

```python
def create_playlist(self):
        database_location = 'postgresql+psycopg2://airflow:airflow@postgres/airflow'
        # spotify_user_id = Variable.get("SPOTIFY_CLIENT_ID")
        # access_token = token.access_token

        headers = {
            "Accept" : "application/json",
            "Content-Type" : "application/json",
            "Authorization" : "Bearer {}".format(self.access_token)
        }

        # Create a new playlist
        print("Trying to create playlist...")

        today = date.today()
        todayFormatted = today.strftime("%d/%m/%Y")


        query = "https://api.spotify.com/v1/users/{}/playlists".format(self.user_id)

        request_body = json.dumps({
            "name": "AI Generated Playlist - {}".format(todayFormatted),
            "description": "An AI generated playlist based on the previous week's listening history",
            "public": True
            }
        )

        response = requests.post(query,
                                 data=request_body,
                                 headers=headers
        )

        response_json = response.json()
        print(response_json)
        return response_json["id"]
```


```python
def add_to_playlist(self, uris: list):
        # add all songs to new playlist
        print("Adding songs...")

        self.new_playlist_id = self.create_playlist()

        uris_str = ','.join(uris)
        print(uris_str)
        api_arg = "https://api.spotify.com/v1/playlists/{}/tracks?uris={}".format(self.new_playlist_id, uris_str)
        # api_arg = "https://api.spotify.com/playlists/{}/tracks".format(self.new_playlist_id) #, uris_str)

        response = requests.post(api_arg,
                                #  data={"uris": uris},
                                 headers={"Content-Type": "application/json",
                                          "Authorization": "Bearer {}".format(self.access_token)
                                          }
        )
        print("Finished - Response JSON:", response )
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
