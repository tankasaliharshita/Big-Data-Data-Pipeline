# Data Pipeline for Live Lacrosse Gamestream

Introduction

This project is designed to work with data in a distributed environment. The main is goal of the project is to work on a simulated lacrosse game stream, and a database of player and team reference data and create a data pipeline that processes the game stream, creates a box score and updates the database tables when the game is over.

# The Problem

The objective is to create a data pipeline which processes a simplified version of an in-game stream from a simulated a lacrosse game. The game stream has been simplified to only process goals scored. There are two parts to this problem:

* At any point while the game is in progress, the game stream should be converted into a JSON format so the web developers can use it to create a box score page on a website. This JSON should be written to the mongodb/sidearm/boxscores collection, and should contain all the data necessary to display the box score page from a single query to the database.
  
* When the game is over, the player and team reference data should be updated to reflect the team records and player statistics after the competition has ended. Normally you would update the mssql tables, but for this project I will create new tables with the updated data, players2 and teams2 respectively. This is mostly because spark does not support row-level updates.

# The Environment Setup and Tools Used

A environment which has a docker-compose.yaml file that stimulates the distributed environment, it consists of the following services:

Databases:

* mssql - A Microsoft SQL Server database that stores the player and team reference data. The database is called sidearmdb and the tables are players and teams.
* minio - An S3 compatible object store that contains the live game stream. The game stream is stored in the minio/gamestreams bucket.
* mongodb - A mongodb database that stores the game stream's real-time box score so the web developers can create a page from the data. The box score is written to the mongodb/sidearm/boxscores collection.

Tools:

* drill - An instance of Apache Drill that can be used to query the databases. The drill-storage-plugins folder contains the configuration files for the databases. 
* jupyter - An instance of Jupyter Lab that can be used to write PySpark code.

Scripts:

* gamestream - A python script that simulates a lacrosse game stream. As game events happen, it writes the game stream to a file in the minio/gamestreams S3 bucket.

# Managing the environment

Docker Containers: mssql, mongodb, jupyter, minio, gamestream

* To check if all the containers are running (docker commands in command line):

PS > docker-compose ps

* To STOP all the containers are running (docker commands in command line):

PS > docker-compose down OR PS > docker-compose stop

Starting and stopping the environment

* To start the environment, run the following commands:

Start the databases: PS> docker-compose up -d mssql mongodb minio

Make sure the databases are running: PS> docker-compose ps

Start the tools: PS> docker-compose up -d drill jupyter

* Finally, start the gamestream: PS> docker-compose up -d gamestream

Make sure the gamestream is running by looking at the logs: PS> docker-compose logs gamestream

Valid gamesteam output looks like this:

| Added s3 successfully. 

| Bucket created successfully s3/gamestreams.

| Bucket created successfully s3/boxscores. 

| Commands completed successfully. 

| Commands completed successfully. 

| INFO:root:Waiting for services... 

| INFO:root:Bucket exists...ok 

| INFO:root:Starting Game Data Stream. Delay: 1 second == 0.25 seconds. 

| INFO:root:Wrote gamestream.txt to bucket gamestreams at 59:51

* To stop the environment, run the following command: PS> docker-compose down

* To start over from the very beginning (erase the volumes) run the following command: PS> docker-compose down -v

# Managing the gamestream

The gamestream container simulates the live game. Each time the game stream is started:

* The players and teams database tables are reset back to their original state.
* The live game is replayed from the beginning, writing events to s3/gamestreams/gamestream.txt as they occur.
* The same game is played each time, with the same game events. The expected behavior will help you write the code.
* Restarting the game steam will NOT erase any other data in mongo or other tables in mssql. See "Start over from the very beginning" if you need to do that.
  
* To Restart the game stream:, run the following command:

PS> docker-compose restart gamestream

To view the gamestream activity, run the following command:

PS> docker-compose logs gamestream

* Adjusting the gamestream speed

By default the game stream "plays" at 4x speed. That 0.25 seconds of real time is 1 second of game time. You can adjust this by setting the DELAY environment variable in the .env file. DELAY=1 plays the game in real time, and DELAY=0.1 plays the game at 10x speed. If you adust the DELAY environment variable, you will need to rebuild the gamestream container. To do this, run the following commands:

PS> docker-compose stop gamestream

PS> docker-compose rm -force gamestream

PS> docker-compose up -d gamestream

NOTE: You can always docker-compose down everything and bring it back up with docker-compose up -d too.

