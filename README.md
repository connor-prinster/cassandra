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

## Exercise 3
* View metadata
```sql
DESCRIBE TABLE videos
```

* What is the partition key?: Partition key is the first column. All rows containing the partition key are stored on the same physical node. insertion/update/delete on rows with same partition key are performed atomically.
* Clustering columns: clustering key groups columns in a row. You can have multiple cluster key groups
* info above taken heavily from [this StackOverflow answer](https://stackoverflow.com/questions/24949676/difference-between-partition-key-composite-key-and-clustering-key-in-cassandra#:~:text=In%20CQL%2C%20the%20order%20in,on%20the%20same%20physical%20node.)

* write a CREATE TABLE statment that will store data partioned by tags
```sql
CREATE TABLE videos_by_tag(
    tag text,
    video_id timeuuid, 
    added_date timestamp, 
    title text, 
    PRIMARY KEY((tag, video_id))
);
```

* copy data into videos_by_tag
```sql
COPY videos_by_tag(tag, video_id, added_date, title)
FROM '/home/ubuntu/labwork/data-files/videos-by-tag.csv'
WITH HEADER = TRUE;
```

* Select data tagged with *cassandra*
```sql
SELECT * FROM videos_by_tag WHERE tag='cassandra' ALLOW FILTERING
```
