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

# Exercise 4
* Drop `videos_by_tag` table
* Modify `CREATE TABLE videos_by_tag (tag text,video_id uuid, added_date timestamp, title text, PRIMARY KEY ());` to partition based on the tag. The table should also store the rows of each partiotion so that the newset videos are listed first

```sql
CREATE TABLE videos_by_tag(
    tag text,
    video_id timeuuid,
    added_date timestamp,
    title text,
    PRIMARY KEY((tag), added_date)
)
WITH CLUSTERING ORDER BY (added_date DESC);
```

* Select from videos_by_tag with oldest video first
```sql
select * from videos_by_tag WHERE tag='cassandra' ORDER BY added_date DESC;
```

* restrict partition key to 'cassandra' and retrieve videos made in 2013 or later
```sql
select * from videos_by_tag where tag = 'cassandra' AND added_date >= '2013-01-01 00:00:00.000000+0000';
```

# Exercise 5
```python
from cassandra.cluster import Cluster

# connect to the database
cluster = Cluster(protocol_version = 3)
session = cluster.connect('killrvideo')

# get data (with a formatter)
print('{0:12} {1:40} {2:5}'.format('Tag', 'ID', 'Title'))
for val in session.execute("select * from videos_by_tag"):
    print('{0:12} {1:40} {2:5}'.format(val[0], val[2], val[3]))

# insert into database
session.execute("insert into videos_by_tag (tag, added_date, title, video_id) values ('stuff', '2020-10-16 00:00:00.000000+0000', 'title stuff', 4845ed97-14bd-11e5-8a40-8338255b7e34)")

# delete from database
session.execute("DELETE FROM videos_by_tag where timestamp=2020-10-16 00:00:00.000000+0000';
```
