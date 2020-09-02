# DataStax Cassandra Course
## [Cassandra 201](#Cassandra-201)
## [Cassandra 202](#Cassandra-202)

# Cassandra-201
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

Apache Cassandra™ doesn't have to have an actual partition with a key value to
determine which nodes will store that partition. You can try any partition key value you
like. For example, try the following:
```sh
/home/ubuntu/node1/resources/cassandra/bin/nodetool getendpoints killrvideo
videos_by_tag 'action'
/home/ubuntu/node1/resources/cassandra/bin/nodetool getendpoints killrvideo
videos_by_tag 'horror'
```

# Exercise 13 - Consistency:

* CAP Theorem (distrubuted systems)
    * Consistency: All data in all replicas are identical
    * Availability: make sure everything can see the database at any time (cassandra)
    * Partition Tolerance: have nodes not see each other (cassandra)

* Consistency Level (CL)
    * CL=1
        * only one replica needs to acknowledge read/write
        * Fastest consistency level
    * CL=QUORUM (Strongly consistent) 
        * 51% acknowledges write or agrees on read
    * CL=ALL
        * All nodes must agree on EVERYTHING
        * slowest
        * removing partitions
        * making all nodes into one.

* change where cqlsh connects:
```
nodeX/bin/cqlsh 127.0.0.x 904x
```

# Exercise 14 - Hinted Handoffs
* If a coordinator node attempts to send data to a downed replication node, the coordinator node holds onto the data. Once the replication comes back on, the coordinator node sends the hint again
    * these hints are stored in a file we specify
    * you can disable hinted handoffs
    * default store time is 3 hours. This can be changed.

* Using `CONSISTENCY ANY` allows even a hint to be sufficient. This is not good

* You can see the hints inside the `nodeX/data/hints`. If we do the following command *CONSISTENCY ANY* in cqlsh, and the nodes are down, it will be stored in that `hints` folder until the nodes are back online

# Read Repair
* Read repair always occurs wehn consistency level is set to *ALL*
* `read_repair_chance` sets the probability which Cassandra will perform a read repair with a consistency level less than *ALL*

# Node Sync:
* can set time that we check that all nodes are synced with latest data
```sql
CREATE TABLE myTable (...) WITH nodesync = {'enabled': 'true'}
```

# Exercise 16: Write Path //TO-DO//
* RAM: (MemTable) 
    * Data is stored according to partition order
    * If full, Cassandra *flushes* it down to HDD
* HDD-1: (commit log) 
    * Data is "committed" to the bottom no matter what
    * nuked when MemTable is *flushed*
    * used when a crashded node restarts
* HDD-2: (Storted Stream Table)
    * When commit log is nuked, the data turns into this
    * Sorted Stream Table
        * sorted partition value
        * immutable
* a `WRITE` operation is acknowledged by a client when the **commit log** and the **MemTable** are written to

# Exercise 17: Read Path
* Reads from an SSTable
* The partitions in an SSTable is actually stored on disk because of how big it is.
* Cassandra builds a `Partition Summary` (in RAM) as a list of **byte offsets** inside the partitions. Easier and fewer comparisons.
* Uses something called a `Bloom Filter` which checks if it **might** be there.
* trie-based algorithm?

# Exercise 18: Compaction
* compacts multiple partitions in multiple partitions
    * will always choose the latest timestamp 
    * if combination of tables has a comparision between `Betsy` and `<insert tombstone longer than "grace seconds allowed>`, both are nuked and neither are added to the new SSTable
    * Doesn't matter what the old value is, the newer value will always replace it.
    * If a tombstone is not more than `grace seconds allowed`, but is the latest change, the tombstone will be written to the new SSTable
* in which scenario would a new partition on disk be larger than either of its input partition segments after a compaction?
    * input partition segments are mainly `INSERT` operations
* benefits of compaction:
    * more optimal disk usage
    * faster reads
    * less memory pressure


# Cassandra-202

## Data Modeling
In Cassandra, this is when we start thinking about keys, tables, and columns

## Relational vs Apache Cassandra
* We think of the *Application* before we start thinking about how to model the data (what we need to cluster on in order for the fastest read time).
    * in relational, *Application* is the last thing we worry about
* Cassandra cannot comply with ACID
* Joins are essentially not possible because Cassandra may have unequal tables
* "Visuals" of difference between Relational and Cassandra
    * Relational: `Data -> Model -> Application`
    * Cassandra: `Application -> Model -> Data`
* ACID

    | Term | Definition | Cassandra Fails it
    | --- | --- | --- | 
    | Atomicity | All statements in a transaction succeed (commit) or none of them do (rollback)
    | Consistency | Transactions cannot leave database in inconsistent state. New database state must satisfy integrity constraints
    | Isolation | Transactions do not interfere with each other
    | Durabilility | completed transactions persist in the event of subsequent failure


* The reason Cassandra doesn't enforce referential integrity is that it would require a read before a write.
    * it falls on the developer's side to adjust/fix that.

# More CQL
* keyspaces: contain table
* tables: contains data
* basic data types
    * text/Ascii/Varchar
    * uuid
    * timestamp
    * int
    * timeUUID    
    * Blob: arbitrary bytes
    * Boolean
    * Counter (only one counter column per table)
    * Inet: IP address
* import CSV
* USE: select keyspace
* SELECT: select rows based on certain data
* TRUNCATE: wipe data from a table. Keeps the schema though
* ALTER TABLE: can change schema EXCEPT for the primary key
    * ADD column
    * DROP ccolumn
* SOURCE: execcutes a file containing CQL file
    * SOURCE `'./myscript.cql';`


# Data modeling
* Partition: 
    * row(s) of data
* Partition key:
    * defines the node on which the data is stored
    * Partition key determines which node the partition resides on
* Row
    * one or more CQL columns stored together on a partition
* Column
    * similar to a relational column
    * primary key
        * used to access data in a table and guarantees uniqueness
* Clustering Column
    * defines the order of rows *within* a partition

# Querying
* MUST use all fields that make up a `PRIMARY KEY` in order to query a database.

# Clustering Columns
* How Cassandra sorts data within each partition
* Sort of example
* PRIMARY KEY((id)) vs PRIMARY KEY((year), title)
    1. three different partitions
    2. Two partitions based on `year`.
* If we do not specify `WITH CLUSTERING ORDER` in a table like below, it will automatically sort `ASC`
    ```sql
    CREATE TABLE videos(
        ...
        PRIMARY KEY ((year), name)
    )
    WITH CLUSTERING ORDER BY (name DESC);
    ```

# Denormalizing
* joins cannot be done on Cassandra tables

# Table Features
* Can use UDTs, or User Defined Types
* Can use counters in one column of each table.

# Data Scructures
* Collections:
    * multi-valued columns
    * SET
        * stored unordered
        * retrieved in sorted
        * Example:
        ```sql
            CREATE TABLE users (
                emails set<text>
            );
        ```
    * LIST
        * stored in order
        * example
        ```sql
        ALTER TABLE users ADD freq_dest list<text>;
        ```
    * MAP
        * key-value pair
            * both with type
        *  example
        ```sql
        ALTER TABLE users ADD todo map<timestamp, text>;
        ```
    * FROZEN    
        * nest datatypes
        * serialize multiple components into a single value
        * treated like blobs
        * example
            ```sql
            ALTER TABLE videos ADD encoding list<frozen<video_encoding>>;
            ```

# UDTs
* group related fields of information that are named and typed
* can contain any supported type
* definition example   
    ```sql
    CREATE TYPE address (
        street text,
        city text,
        zip_code int,
        phones set<text>
    );

    CREATE TYPE full_name (
        first_name text,
        last_name text
    )
    ```
* usage example
    ```sql
    CREATE TABLE users (
        id uuid,
        name frozen <full_name>,
        direct_reports set<frozen <full_name>>,
        addresses map<text, frozen<address>>,
        PRIMARY KEY ((id))
    );
    ```

# Counters
Example of how to create a counter in a table
```sql
CREATE TABLE moo_counts (
    cow_name text,
    moo_count counter,
    PRIMARY KEY((cow_name))
);

UPDATE moo_counts
SET moo_count = moo_count + 8;
WHERE cow_name = 'Betsy';
```

* Counter considerations:
    * distributed systemcs can cause consistency issues with counters
    * cannot INSERT or assign values. Default value is 0
    * Must be only non-primary key column(s)
    * Not idempotent
    * Must use UPDATE command (dse rejects using TIMESTAMP or TTL to update counter columns)
    * Counter columns cannot b eindexed or deleted