## Concept
For this project I am working for the startup company, Sparkify. They have been collecting data on songs and user acivity with their new music streaming app. They currently dont have a good method to query the data they need which resides in JSON logs and metadata for user activity and songs. This will help them understand what songs users are listening to. 

This is where I come in as a data engineer. My task is to create a Apache Casandra database with tables designed to optimize queries on song play analysis. 

In this repository I have created a database schema and ETL pipeline using python for analysis. 

## Content
- The Data
- Creating Tables, ETL Pipeline, and Quality Check
- Run

## The Data
For this project, you'll be working with one dataset: event_data. The directory of CSV files partitioned by date. Here are examples of filepaths to two files in the dataset:

event_data/2018-11-08-events.csv
event_data/2018-11-09-events.csv

The first part of this process is to get our data into a readable version so that we are able to process it. We want to process all data files and log them in a .csv file to be  processed from there. 

    # creating a smaller event data csv file called event_datafile_full csv that will be used to insert data into the \
    # Apache Cassandra tables
    csv.register_dialect('myDialect', quoting=csv.QUOTE_ALL, skipinitialspace=True)

    with open('event_datafile_new.csv', 'w', encoding = 'utf8', newline='') as f:
        writer = csv.writer(f, dialect='myDialect')
        writer.writerow(['artist','firstName','gender','itemInSession','lastName','length',\
                    'level','location','sessionId','song','userId'])
        for row in full_data_rows_list:
            if (row[0] == ''):
                continue
            writer.writerow((row[0], row[2], row[3], row[4], row[5], row[6], row[7], row[8], row[12], row[13], row[16]))
            
the above code outputs a file named 'event_datafile_new.csv':

!(image1)[images/image_event_datafile_new.jpg]

## Creating Tables, ETL Pipeline, and Quality Check

When creating tables with a NOSQL database it is import too know what queries you are going to implement. We want tables to be specific to the query that we search. The queries I selected are as follows:

    1. Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4
    2. Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182
    3. Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'

### Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4
Because we are simply creating a WHERE clause without the need for ordering I set the Partition Key as session_id and  Clustering Key as item_in_session:

    ## TO-DO: Query 1:  Give me the artist, song title and song's length in the music app history that was heard during \
    ## sessionId = 338, and itemInSession = 4
    query = "CREATE TABLE IF NOT EXISTS music_history"
    query = query + "(artist_name text, song_title text, session_id int, item_in_session int, PRIMARY KEY (session_id, item_in_session))"
    try:
        session.execute(query)
    except Exception as e:
        print(e)
        
Create a query to get our data:

    ## TO-DO: Add in the SELECT statement to verify the data was entered into the table
    query = """select artist_name, song_title from music_history 
            WHERE session_id=338 
            AND item_in_session=4"""
    try:
        rows = session.execute(query)
    except Exception as e:
        print(e)

    for row in rows:
        print (row.artist_name, " | ", row.song_title)
        
Output:

    Faithless | Music Matters (Mark Knight Dub)
    
For the purpose of showing how our query is working I would like to show the session_id and item_in_session to demonstrate how the partition and clustering keys are working. The Partition key is what tells the database to partition each node by, and in that partion we want to order by the clustering key. This allows for faster reads of the data. so with our partion key as session_id and the clustering key as item_in_session we get the following data:

    247  |  0  |  Discovery  |  I Want You Back
    247  |  1  |  Justin Bieber  |  Somebody To Love
    247  |  2  |  Rancid  |  Hooligans (Album Version)
    796  |  0  |  Florence + The Machine  |  Kiss With A Fist
    919  |  0  |  Des'ree  |  Life
    117  |  1  |  Vangelis  |  La Petite Fille De La Mer
    117  |  2  |  Justin Bieber  |  Stuck In The Moment
    117  |  3  |  Sleep  |  Evil Gypsy - Solomons Theme
    117  |  4  |  Leon Gieco  |  La Colina De La Vida
    117  |  5  |  George Jones and Gene Pitney  |  Wreck On the Highway
    117  |  6  |  Barry White  |  Come On
    117  |  7  |  Justin Bieber  |  Love Me
    117  |  8  |  Darude  |  Sandstorm
    117  |  9  |  Evanescence  |  My Immortal
    117  |  10  |  Harmonia  |  Sehr kosmisch
    
### Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182
The second involves ordering the data so the primary key order in this case is crucial. We must create a Composite key which will partion by user_id, and have and oder of session_id and order that partition by item_in_session. This will allow the songs titles to be ordered by item_in_session:

    query = "CREATE TABLE IF NOT EXISTS user_history"
    query = query + """(artist_name text, song_title text, item_in_session int, first_name text, last_name text, user_id int, session_id int,
                    PRIMARY KEY ((user_id, session_id), item_in_session)
                    );"""
    ...
    
Query to get data:

    query = """SELECT * FROM user_history
            WHERE user_id=10
            AND session_id=182"""
 
Output:

    Down To The Bone  |  Keep On Keepin' On  |  Sylvie  |  Cruz
    Three Drives  |  Greece 2000  |  Sylvie  |  Cruz
    Sebastien Tellier  |  Kilometer  |  Sylvie  |  Cruz
    Lonnie Gordon  |  Catch You Baby (Steve Pitron & Max Sanna Radio Edit)  |  Sylvie  |  Cruz
    
lets see how our query works by selecting all data inserted into the table to see how the primary key works. As mentioned before, we have a partion of user_id, ordered by session_id and that cluster is ordered by item_in_session:

    10  |  182  |  0  |  Keep On Keepin' On  |  Down To The Bone  |  Sylvie  |  Cruz
    10  |  182  |  1  |  Greece 2000  |  Three Drives  |  Sylvie  |  Cruz
    10  |  182  |  2  |  Kilometer  |  Sebastien Tellier  |  Sylvie  |  Cruz
    10  |  182  |  3  |  Catch You Baby (Steve Pitron & Max Sanna Radio Edit)  |  Lonnie Gordon  |  Sylvie  |  Cruz

### Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'
The last was simple enough to just include a Simple Key, the song_title:

    query = "CREATE TABLE IF NOT EXISTS play_history"
    query = query + """(first_name text, last_name text, song_title text,
                        PRIMARY KEY (song_title)
                        );"""
                        
Query to get data:

    query = """SELECT * FROM play_history
            WHERE song_title = 'All Hands Against His Own'"""
            
Output:

    Sara  |  Johnson  |  All Hands Against His Own
    
## Run
I am running this program on a jupyter lab and should work with all versions (currently May 2020).

In order to successfully run the program You must have all files downloaded.

- download and install latest Apache cassandra
- Run Pipeline.ipynb (creates required tables, processes data to database, and tests if data is inserted correctly, drops tables in the end)
