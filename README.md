# Big-Data-Data-Pipeline
INTRODUCTION 

This project is designed to work with data in a distributed environment. The main is goal of the project is to work on a simulated lacrosse game stream, and a database of player and team reference data and create a data pipeline that processes the game stream, creates a box score and updates the database tables when the game is over.

# The Environment Setup

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

# Starting and stopping the environment

To start the environment, run the following commands:

* Start the databases:

PS> docker-compose up -d mssql mongodb minio
Make sure the databases are running

PS> docker-compose ps

* Start the tools:

PS> docker-compose up -d drill jupyter

Make sure the tools are running

PS> docker-compose ps

* Finally, start the gamestream:

PS> docker-compose up -d gamestream

* Make sure the gamestream is running by looking at the logs:

PS> docker-compose logs gamestream

* Valid gamesteam output looks like this:
  
  | Added `s3` successfully.
  
  | Bucket created successfully `s3/gamestreams`.
  
  | Bucket created successfully `s3/boxscores`.
  
  | Commands completed successfully.
  
  | Commands completed successfully.
  
  | INFO:root:Waiting for services...
  
  | INFO:root:Bucket exists...ok
  
  | INFO:root:Starting Game Data Stream. Delay: 1 second == 0.25 seconds.
  
  | INFO:root:Wrote gamestream.txt to bucket gamestreams at 59:51
  
IF YOU DON'T SEE THIS, THE SERVICE IS NOT RUNNING.

* To stop the environment, run the following command:
PS> docker-compose down

* To start over from the very beginning (erase the volumes) run the following command:
PS> docker-compose down -v

# Managing the gamestream
The gamestream container simulates the live game. Each time the game stream is started:

* The players and teams database tables are reset back to their original state.
* The live game is replayed from the beginning, writing events to s3/gamestreams/gamestream.txt as they occur.
* The same game is played each time, with the same game events. The expected behavior will help you write the code.
* Restarting the game steam will NOT erase any other data in mongo or other tables in mssql. See "Start over from the very beginning" if you need to do that.
  
To Restart the game stream:, run the following command:
PS> docker-compose restart gamestream

To view the gamestream activity, run the following command:
PS> docker-compose logs gamestream

Adjusting the gamestream speed

By default the game stream "plays" at 4x speed. That 0.25 seconds of real time is 1 second of game time. You can adjust this by setting the DELAY environment variable in the .env file. DELAY=1 plays the game in real time, and DELAY=0.1 plays the game at 10x speed. If you adust the DELAY environment variable, you will need to rebuild the gamestream container. To do this, run the following commands:

PS> docker-compose stop gamestream

PS> docker-compose rm -force gamestream

PS> docker-compose up -d gamestream

NOTE: You can always docker-compose down everything and bring it back up with docker-compose up -d too.

# The Problem

The objective is to create a data pipeline which processes a simplified version of an in-game stream from a simulated a lacrosse game. The game stream has been simplified to only process goals scored. There are two parts to this problem:

* At any point while the game is in progress, the game stream should be converted into a JSON format so the web developers can use it to create a box score page on a website. This JSON should be written to the mongodb/sidearm/boxscores collection, and should contain all the data necessary to display the box score page from a single query to the database.
  
* When the game is over, the player and team reference data should be updated to reflect the team records and player statistics after the competition has ended. Normally you would update the mssql tables, but for this project I will create new tables with the updated data, players2 and teams2 respectively. This is mostly because spark does not support row-level updates.
  
# Game Stream

While the game is going on, there is a file called gamestream.txt located in the minio/gamestreams S3 bucket. Each time an in-game event happens, the event is appended to this file. To simplify things, the game stream only reports shots on goal. Here is the format of the file each line is an event and the fields are separated by a space:

0 59:51 101 2 0

1 57:06 101 6 0

2 56:13 205 8 1

3 55:25 101 4 0

Data Dictionary for gamestream.txt

* The first column is the event ID. These are sequential. An event ID of -1 means the game is over.
* The second column is the timestamp of the event in the format mm:ss. This counts down to 00:00. For example the first event occurred 9 seconds into the game.
* The third column is the team ID, indicating team took the shot on goal. In the simulation there are only two teams, 101 and 205.
* The fourth colum is the jersey number of the player who took the shot.
* The final column is a 1 if the shot was a goal, 0 if it was a miss.
  
Player and Team Reference Data
The player and team reference data is stored in a Microsoft SQL Server database. The database is called sidearmdb . The database has two tables, players and teams with the following schemas, respectively:

CREATE TABLE teams (

    id int primary key NOT NULL,

    name VARCHAR(50) NOT NULL, 
    
    conference VARCHAR(50) NOT NULL,
    
    wins INT NOT NULL,
    
    losses INT NOT NULL,
    
)

CREATE TABLE players (

    id int  primary key NOT NULL,

    name VARCHAR(50) NOT NULL,
    
    number varchar(3) NOT NULL,

    shots INT NOT NULL,
    
    goals INT NOT NULL,
    
    teamid INT foreign key references teams(id) NOT NULL,
    
)

The teams table, only has two teams, 101 = syracuse and 205 = johns hopkins. Each team has a conference affiliation, and a current win / loss record.

The players table has 10 players for each team. Each player has a name, jersey number, shots taken, goals scored, along with their team id.

# The game stream's real-time box score

Each time you run your code while the game is ongoing, you should write a new boxscore document to the mongodb/sidearm/boxscores collection. That way sidearm web developers can read the latest document's contents to render a webpage for live box score stats while the game is going on.

For simplicity, assume team 101 is the home team and team 205 is the away team.

The document should have the following structure (consider this an example)

{
    "_id" : "UseTheEventIDFrom gamestream.txt",
    "timestamp" : "55:25",
    "home": {
        "teamid" : 105,
        "conference" : "ACC",
        "wins" : 5,
        "losses" : 2,
        "score" : 3,
        "status" : "winning",
        "players": [
            {"id": 1, "name" : "sam",  "shots" : 3, "goals" : 1, "pct" : 0.33 },
            {"id": 2, "name" : "sarah",  "shots" : 0, "goals" : 0, "pct" : 0.00 },
            {"id": 3, "name" : "steve",  "shots" : 1, "goals" : 1, "pct" : 1.00 },
            ...
        ]
    },
    "away": { ... }
}

NOTES:

* "status" should be "winning", "losing" or "tied" based on the current home.score and away.score
* the "_id" should be the latest event ID from the game stream, at the time the box score was written. 
* the "timestamp" should be the timestamp from the game stream.
* Every player on the roster (in the players table) should appear in the box score.
* The stats in the box score should be the current stats for the player in game only, and not include the stats in the players table and add up the shot and goal for every player at that point in the game stream.
* Calculate the pct field
* game is over when the clock hits 00:00.
  
# Updating stats in the database when the game is over

After the game is complete, the tables in the mssql sidearmdb database should be updated, based on the final box score. Specifically:

* update the win/loss record for each team in the teams table
* update the shots and goals for each player in the players table

NOTES:

We will not update the actual tables. Instead we will create new tables called teams2 and players2 with the updated data. It's anti-big data to perform row-level updates. The proper way to move the updates into the original tables would be to write an MSSQL script to update the tables. 
