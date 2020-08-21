# DataStax Cassandra Course

## Exercise 2
Data is organized as below

Column name | Data type
--- | ---
video_id | timeuuid
added_date | timestamp
Title | Text

* Create keyspace: 
```sql
CREATE KEYSPACE IF NOT EXISTS killrvideo WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 1};
```

* Enter keyspace: 
```sql
USE killrvideo
```

* Create table: 
```sql
CREATE TABLE videos(video_id timeuuid PRIMARY KEY, added_date timestamp, Title text);
```

* Insert into table: 
```sql
INSERT INTO videos ("video_id", "added_date", "title") VALUES (1645ea59-14bd-11e5-a993-8138354b7e31, '2014-01-29', 'Cassandra History');
```

* Select from table: 
```sql
SELECT * FROM videos;
```

* Remove all data you inserted: 
```sql
TRUNCATE videos;
```

* Copy from a file: 
```sql
COPY videos(video_id, added_date, title) FROM '/home/ubuntu/labwork/data-files/videos.csv' WITH HEADER=TRUE;
```

* Count all the rows in the keyspace
```sql
SELECT COUNT(*) FROM videos
```

* Exit cql
```sql
EXIT
```