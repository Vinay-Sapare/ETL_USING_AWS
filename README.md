# ETL USING AWS

## Introduction
In this project, we will develop an ETL pipeline leveraging the Spotify API on AWS. Our goal is to extract data from the Spotify API, transform it into a suitable format, and then load it into a Snowflake data warehouse for analysis.
## Services Used
  - **S3 (Simple Storage Service):** It is a cloud storage solution that allows to store and retrieve any amount of data from anywhere on the web. It’s designed to be highly scalable, secure, and easy to use, making it ideal for storing files, backups, and data for applications.
  - **AWS Lambda:** AWS Lambda is a serverless computing service that allows to run code in response to events without managing servers. Lambda automatically handles the execution, scaling, and infrastructure management, enabling you to focus on building applications instead of worrying about the underlying infrastructure.
  - **Cloud Watch:** Amazon Cloud Watch is a monitoring service for AWS resources and the applications you run on them. You can use Cloud Watch to collect and track metrics, collect and monitor log files and for setting alarms.
  - **Snowflake:** It is a cloud-based data warehousing platform that enables organizations to store, manage, and analyze large volumes of data. It allows users to run queries and perform data analysis quickly and efficiently, all while providing seamless scalability and support for diverse data types.
## Project Execution Flow
Extracting Data from Spotify API -> Trigger Lambda Function (every 5 hour) -> Run extract code -> Store raw data in S3 -> Trigger transformation function -> Transform data -> Load into Snowflake -> Run Queries on top of it.
## Architecture
![Architecture Diagram](https://github.com/Vinay-Sapare/ETL_USING_AWS/blob/main/etl_using_aws_architecture.png)

## Process Flow
  - **Extracting the data from Spotify API:** Extract the unstructured data from Spotify's API using proper authentication methods for further processing.
  - **Spotify API Integration:** Configure the Spotify API to retrieve albums, artists, and tracks, and then save the raw data to the data/raw/ directory. This process ensures that the data is collected accurately for subsequent processing.
  - **Deploy code to AWS Lambda:** Develop and deploy a Lambda function that utilizes Spotipy and Boto3 to extract data from Spotify and store it in an S3 bucket, automating the data extraction process in the cloud.
  - **Add Trigger for Automatic Extraction:** Configure AWS CloudWatch Events to automatically trigger the Lambda function at regular intervals, such as daily, enabling continuous data extraction. This setup allows the pipeline to operate on schedule without requiring manual input.
  - **Write Transformation Function** Create Python transformation functions with Pandas to clean and format the raw data, preparing it for analysis. Save the transformed data for future use or upload it to the cloud.
  - **Store Files on S3:** Structure raw and processed data in organized S3 directories (raw_data/ and processed_data/) to ensure clear version control and easy access. Set up the necessary permissions for secure data storage.
  - **Build analytics using Snowflake:** Set up a Snowflake integration to catalog the S3 data and enable SQL-based querying. This facilitates efficient analytics and reporting on the processed data.

## Extraction Code
```
import json
import os
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import boto3
from datetime import datetime

def lambda_handler(event, context):
    
    client_id=os.environ.get('client_id')
    client_secret=os.environ.get('client_secret')
    
    client_credentials_manager = SpotifyClientCredentials(client_id=client_id, client_secret=client_secret)
    
    sp = spotipy.Spotify(client_credentials_manager = client_credentials_manager)
    
    playlists=sp.user_playlists('spotify')
    
    playlist_link="https://open.spotify.com/playlist/3OK8UdRoB6IfDEa1pNJTQE"
    playlist_url=playlist_link.split('/')[-1]
    
    spotify_data=sp.playlist_tracks(playlist_url)
    
    client=boto3.client('s3')
    
    filename="spotify_raw_"+str(datetime.now())+".json"
    
    client.put_object(
        Bucket="spotifyv",
        Key="raw-data/to_processed/"+filename,
        Body=json.dumps(spotify_data)
        )

```

## Transformation Code
```
import json
import boto3
from datetime import datetime
from io import StringIO
import pandas as pd 


def album(data):
    album_list=[]
    for row in data["items"]:
        album_id=row["track"]["album"]["id"]
        album_name = row['track']['album']['name']
        album_release_date = row['track']['album']['release_date']
        album_total_tracks = row['track']['album']['total_tracks']
        album_url = row['track']['album']['external_urls']['spotify']
        album_element = {'album_id':album_id,'name':album_name,'release_date':album_release_date,'total_tracks':album_total_tracks,'url':album_url}
        album_list.append(album_element)
    return album_list
    
def artist(data):
    artist_list = []
    for row in data['items']:
        for key, value in row.items():
            if key == "track":
                for artist in value['artists']:
                    artist_dict = {'artist_id':artist['id'], 'artist_name':artist['name'], 'external_url': artist['href']}
                    artist_list.append(artist_dict)
    return artist_list
    
def songs(data):
    song_list = []
    for row in data['items']:
        song_id = row['track']['id']
        song_name = row['track']['name']
        song_duration = row['track']['duration_ms']
        song_url = row['track']['external_urls']['spotify']
        song_popularity = row['track']['popularity']
        song_added = row['added_at']
        album_id = row['track']['album']['id']
        artist_id = row['track']['album']['artists'][0]['id']
        song_element = {'song_id':song_id,'song_name':song_name,'duration_ms':song_duration,'url':song_url,
                        'popularity':song_popularity,'song_added':song_added,'album_id':album_id,
                        'artist_id':artist_id
                       }
        song_list.append(song_element)
        
    return song_list

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    Bucket = "spotifyv"
    Key = "raw-data/to_processed/"
    
    spotify_data = []
    spotify_keys = []
    for file in s3.list_objects(Bucket=Bucket, Prefix=Key)['Contents']:
        file_key = file['Key']
        if file_key.split('.')[-1] == "json":
            response = s3.get_object(Bucket = Bucket, Key = file_key)
            content = response['Body']
            jsonObject = json.loads(content.read())
            spotify_data.append(jsonObject)
            spotify_keys.append(file_key)
            
    for data in spotify_data:
    
        album_list = album(data)
        artist_list = artist(data)
        song_list = songs(data)
           
        album_df = pd.DataFrame.from_dict(album_list)
        album_df = album_df.drop_duplicates(subset=['album_id'])

        artist_df = pd.DataFrame.from_dict(artist_list)
        artist_df = artist_df.drop_duplicates(subset=['artist_id'])

        song_df = pd.DataFrame.from_dict(song_list)

            
        songs_key = "transformed-data/songs_data/songs_transformed_" + str(datetime.now()) + ".csv"
        song_buffer=StringIO()
        song_df.to_csv(song_buffer, index=False)
        song_content = song_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=songs_key, Body=song_content)
            
        album_key = "transformed-data/album_data/album_transformed_" + str(datetime.now()) + ".csv"
        album_buffer=StringIO()
        album_df.to_csv(album_buffer, index=False)
        album_content = album_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=album_key, Body=album_content)
            
        artist_key = "transformed-data/artist_data/artist_transformed_" + str(datetime.now()) + ".csv"
        artist_buffer=StringIO()
        artist_df.to_csv(artist_buffer, index=False)
        artist_content = artist_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=artist_key, Body=artist_content)
            
    s3_resource=boto3.resource('s3')
    for key in spotify_keys:
        copy_source={
            'Bucket':Bucket,
            'Key':key
            
        }
        s3_resource.meta.client.copy(copy_source,Bucket,'raw-data/processed/'+key.split("/")[-1])
        s3_resource.Object(Bucket, key).delete()
```

## Load Data into Snowflake Data warehouse
```
#create a database 
CREATE DATABASE spotify_data;

#creat tables
CREATE TABLE album(
    id VARCHAR(256),
    name VARCHAR(256),
    release_date DATE,
    total_tracks INTEGER,
    url VARCHAR(256)
   
)

CREATE TABLE artist (
    id VARCHAR(256),
    name VARCHAR(256),
    href VARCHAR(256)
)

CREATE TABLE song (
    id VARCHAR(256),
    name VARCHAR(256),
    duration INTEGER,
    url VARCHAR(256),
    popuarity INTEGER,
    add_date DATE,
    album_id VARCHAR(256),
    artist_id VARCHAR(256)
)

# create a file format #A file format defines how data files are structured and how to interpret the data within them
CREATE OR REPLACE FILE FORMAT csv_format
    TYPE='CSV'
    FIELD_DELIMITER=','
    SKIP_HEADER=1

#create a Stage #A stage is a temporary storage area used to store data files before loading them into a database table
CREATE STAGE aws_stage
    URL='s3://spotifyv'
    CREDENTIALS=(AWS_KEY_ID='AWS_ACCESS_KEY',
                AWS_SECRET_KEY='AWS SECRET KEY')

LIST @aws_stage/transformed-data # LIST command is used to show the files or folder in the stage

#copy the data from stage to album table #The COPY command is used to load data into a table from a stage (either internal or external) or to unload data from a table to a stage
COPY INTO album
FROM '@aws_stage/transformed-data/album_data'
FILE_FORMAT=(FORMAT_NAME='csv_format');

#copy the data from stage to artist table
COPY INTO artist
FROM '@aws_stage/transformed-data/artist_data'
FILE_FORMAT=(FORMAT_NAME='csv_format')

#copy the data from stage to song table
COPY INTO song
FROM '@aws_stage/transformed-data/songs_data'
FILE_FORMAT=(FORMAT_NAME='csv_format')

```

## Conclusion

This project establishes a comprehensive, automated data pipeline that integrates the Spotify API, AWS Lambda, and S3 to facilitate efficient data extraction, transformation, and storage. By leveraging Snowflake for data warehousing, it empowers robust analytics and querying capabilities for the processed data, enabling insightful decision-making and enhanced data-driven strategies.





  
