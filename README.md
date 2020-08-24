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
# as long as you've run 'sudo pip install cassandra-driver', you're golden here
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

# Exercise 6
## ./nodetool flags
command | what it does
--- | ---
help | shows all options
status | shows information about entire cluster and the state of each node
info | shows information about connected node
describecluster | shows settings that are common across all nodes in the cluster
getlogginglevels | shows how each issue/situation will be logged
setlogginglevel | dynamically changes the logging level without the need of a sestart. 
settraceprobability | decimal describing percentage of queries being saved. It is saved in the `system_traces` keyspace
drain | stops writes from occuring on node and flushes data to disk
stopdaemon | stops an execution
flush | writes all written data to disk. It allows further writes to occur

## CQLSH Stuff
* To see current keyspaces
```sql
DESCRIBE KEYSPACES;
```
* current tables
```sql
DESRIBE TABLES
```

# Exercise 7

Start up node1 and node2

```sql
CREATE KEYSPACE killrvideo
WITH replication = {'class': 'SimpleStrategy',
'replication_factor': 1 };
USE killrvideo;
CREATE TABLE videos (
 id uuid,
 added_date timestamp,
 title text,
 PRIMARY KEY ((id))
);
COPY videos(id, added_date, title)
FROM '/home/ubuntu/labwork/data-files/videos.csv'
WITH HEADER=TRUE;
CREATE TABLE videos_by_tag (
 tag text,
 video_id uuid,
 added_date timestamp,
 title text,
 PRIMARY KEY ((tag), added_date, video_id))
 WITH CLUSTERING ORDER BY(added_date DESC);
COPY videos_by_tag(tag, video_id, added_date, title)
FROM '/home/ubuntu/labwork/data-files/videos-by-tag.csv'
WITH HEADER=TRUE;
```

Running `/home/ubuntu/resources/cassandra/bin/nodetool/getendpoints killrvideo videos_by_tag 'cassandra'` will return the IP addresses of the node(s) which store the partitions with the key value (*cassandra* or *datastax* in our case). 


# Exercise 9 - VNodes

If you set up `vim /home/ubuntu/node2/resources/cassandra/conf/cassandra.yaml` to have `num_tokens: 128` and comment out `initial_token`, starting up the two nodes will have 128 tokens within both.

Cassandra sets the tokens itself.

# Exercise 10 - Gossip
A Gossip Node starts up the gossip step to make sure that each node has the proper (most up-to-date) information

Gossipping stage runs continuously in the background and does not affect traffic

* each node initiates a gossip round every second
* a node picks 1-3 to gossip to
* can gossip to ANY other node
* Slightly favor seed and downed nodes
* Nodes don't track who they communicated with before
* reliably and efficiently spreads node metadata throughout cluster
* fault tolerant--continues to spread even if nodes fail.

Gossip Metadata (Endpoint State):
* Heartbeat State:
    * generation (timestamp boostrapped)
    * version
* Application State:
    * Status=NORMAL/BOOTSTRAP/LEFT/REMOVE
    * DC (Data Center)
    * Rack
    * SCHEMA
    * Load
    * ...
* A node sends out a SYN message, it receives an ACK message.
* If ACK message tells original node it is out of date, it sends an ACK2 message to second node. 
    * The second node responds to the ACK2 with an updated ACK2 containing the proper data

To check gossip information on the cluster, run this
```sh
/home/ubuntu/node1/resources/cassandra/bin/nodetool gossipinfo
```

# Exercise 11 - Snitch

Some snippets from the exercise

* Edit /home/ubuntu/node1/resources/cassandra/conf/cassandra.yaml and
find the endpoint_snitch setting.
NOTE: Using endpoint_snitch default DseSimpleSnitch places your node in a
datacenter that is based upon work type
```yml
# You can use a custom Snitch by setting this to the full class name
# of the snitch, which will be assumed to be on your classpath.
endpoint_snitch: com.datastax.bdp.snitch.DseSimpleSnitch
```
* Change the endpoint_snitch to GossipingPropertyFileSnitch.
```yml
# You can use a custom Snitch by setting this to the full class name
# of the snitch, which will be assumed to be on your classpath. 
endpoint_snitch: GossipingPropertyFileSnitch
```

* Edit /home/ubuntu/node1/resources/cassandra/conf/cassandrarackdc.properties file.
```yml
# These properties are used with GossipingPropertyFileSnitch and will
# indicate the rack and dc for this node
dc=dc1
rack=rack1
```

/home/ubuntu/node1/bin/dsetool status
Notice the differing datacenter assignments now.

```yml
DC: east-side Workload: Cassandra Graph: no
======================================================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
-- Server ID Address Load Owns VNodes Rack Health [0,1]
UN 06-74-00-EF-3F-E8 127.0.0.2 142.47 KiB ? 128 hakuna-matata 0.20
DC: west-side Workload: Cassandra Graph: no
======================================================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
-- Server ID Address Load Owns VNodes Rack Health [0,1]
UN 06-74-00-EF-3F-E8 127.0.0.1 117.54 KiB ? 128 hakuna-matata 0.20 
```

# Replication

* multiple nodes is responsible for certain range of data based on partitioner

### Examples
* RF=1
    * each node stores only its own data
    * if data comes into the wrong partition, that node shoves it off to the correct partition
* RF=2
    * each node tracks its own data AND one of its neigbhors' data
    * if data comes to wrong partition, the data is copied to both nodes holding the data.
* RF=3
    * typically a good replication factor
    * It's a good chance that all data will remain up, even if nodes go down

### Node Goes Down
* if two nodes go down, a replication factor has already spread its data across **n=RF** nodes. If someone tries to write to the database, it PROBABLY has a home for it.

### Multiple datacenters
* snitch makes sure that it gets to both datacenters

# Exercise 12 - Replication