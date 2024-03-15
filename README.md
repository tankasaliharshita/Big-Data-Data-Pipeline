# Big-Data-Data-Pipeline

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

