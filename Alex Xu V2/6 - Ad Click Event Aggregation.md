# TLDR
* Mostly Event Aggregations, like Metric / Monitoring
* Scale using brokers
* Have Raw Database and Aggregate Database
* Message queues to ensure delivery and exactly once
* Have aggregator and resource manager for servers
* Snapshots in case of failure
* Questions
	* What do the logs look like? What's diff in raw and processed data?
	* DAG to do these calculations?
		* Map Reduce
			* It's a divide and conquer strategy like merge sort
			* You break every data point into pieces and sort the smaller sections and build your way up
			* "Map" means that you hash / shard / separate the data into groups
			* Aggregate means you perform your count operation for each ad_id
			* Reduce means you send only the top 3 into your next node 
		* Nodes are great because we can normalize the data, although it's okay to do this with normal sharding too (You have to normalize after)
		* For OLAP, you can make quick filtering by caching extra results onto the table
			* Instead of querying a ton of rows, just query the column
			* So ad_id_01 in the USA and click minute X ---> has count 100
			* Pre-calculated data

There's big billions of dollars in online advertising. This is the back-bone of the most popular social media apps.

Real-Time Bidding is a core component
* Giving the advertising space to whoever auctions for it (AND has good quality ads)

# Step 1 - Understand Problem and Establish Design Scope

What's the form of the data?
* Log file on different servers
* Latest click events are appended to the end of file
* Has ad_id, click_timestamp, user_id, ip, and country

What's the data volume?
* 1 billion ad clicks per day and 2 million ads in total
* Grows 30% year over year

Important queries to support?
* Return top number of clicks events for a particular ad in the last M minutes
* Return top 100 most clicked ads in the past 1 minute
	* Both are configurable
* Support data filtering by ip, user_id, country

Do we need to worry about edge cases?
* Let's say things like:
	* There might be events that arrive later than expected
	* There might be duplicated events
	* Different parts of the system might be down at any time.
* Yes, let's consider all of those

What's latency requirement?
* Few minutes of end-to-end latency
* RTB and click aggregation are different requirements
	* RTB is les than one second
	* Few minutes for ad-click event aggregation because it's use for billing

### Functional Requirements
* Aggregate the number of clicks of ad_id in last M minutes
* Return top 100 most clicked ad_id every minute
* Support aggregation filtering by different attributes
* Dataset volume is Facebook or Google scale

### Non-Functional Requirements
* Correctness of the aggregation result is important as the data is used for RTB and ads billing
* Properly handle delayed or duplicate events
* Robustness
* Latency Requirement. End to end latency should be a few minutes

#### Back of Envelope Estimations
* 1 Billion DAU
* 1B Ad clicks per day
	* 2M adds total
	* Ad click QPS -> 10^9 / 10^5 -> 10^4
		* 10,000 QPS
		* Peak QPS is 5x so like 50,000
* Storage -> .1KB x 1B every day = 100GB every day
	* 3TB per month

# Step 2 - Propose High Level Design and Get Buy In

#### API 1 -> Aggregate number of click for ad_id in last M Minutes

GET v1/ads/{:ad_id}/aggregated_count | Return aggregated event count for given ad_id

Request parameters are
* from | Start minute
* to | end minute
* filter | filter strategies

Response
* ad_id | identifier for ad
* count | aggregated count

### API 2: Return top N most clicked ad_ids in last M minutes

GET v1/ads/popular_ads

Request Parameters
* count | Top N most clicked ads
* window | M minutes
* filter | filter strategies

Response
* ad_ids | ist of most clicked ads

#### Data Model

Raw data looks like
* ad_id, timestamp, user_id, ip-address

### Aggregated data

ad_id | timestamp | count

Can also add another table for the filter_id to do a JOIN for countries with an ad
#### Should we Store Raw or Aggregated Data
* Raw
	* Pros
		* Full data set
		* Allows filtering
	* Cons
		* Huge data storage
		* Slow query
* Aggregated Data only
	* Pros
		* Smaller data
		* Fast query
	* Cons
		* Data loss

His suggestion -> Do Both
* Keep the raw data as a back up that's rarely used

### Choosing a database

It's write heavy so Cassandra makes sense with NOSQL
SQL is hard to scale
Alternative is AWS S3 with a columnar data format like ORC, Parquet, or AVRO

Aggregated data is time-series in nature. Read and write heavy
* Could use the same database to store both raw and aggregated
### High Level design

![[Pasted image 20240213083652.png]]


It's too easy for data to get lost here and problems to occur
* Load balancing
* Retry mechanisms

Use Message queues

![[Pasted image 20240213090031.png]]

### Aggregation Service

MapReduce framework to aggregate ad click events

Use a DAG (directed acrilyic graph) to have different steps of what to do with the data
* forms a system

![[Pasted image 20240213090339.png]]

Why not just use Kafka?
* Input data may need to be cleaned or normalized
* Can be done by the map node

Aggregate node
* Basically A reduce node
Reduce node
* Reduces the total number of nodes

This is the Map Reduce Paradigm and it quite popuarl

# Design Deep Dive

### Streaming vs. Batching
Type of stream processing system

(Services(online), Batch(offline), Streaming(realtime))

![[Pasted image 20240213090928.png]]

#### Data Recalculation

If there's bugs, sometimes we want to recalculate the aggregated data from raw.
* Retreive from the raw storage (Batch job)
* Send to the dedicated aggregation service so real-time isn't affected
* Aggregated results are sent to teh second message queue, then updated in DB

#### Timestamp

When to generate a timestamp? When a click happens, or when it's process?

Because we're using message queues and disitrubted systems, the latency is large
* Can be 5 hours after an event was clicked

![[Pasted image 20240213091417.png]]


If using an event time as the timestamp, can add a watermark to improve if delays occur

#### Aggregation window

DDIA says 4 types of windows
* Tumbling window, hopping window, session window, sliding window

Tumbling and sliding window will be used in this system
* Look up what Tumbling window is

#### Delivery Guarantees

Aggregation results are used for billing so data accuracy and completeness are important
* How to avoid duplicates?
* How to ensure all events are processed?

Message Queues 
* Kafka provides three delivery semantics
* At-most once, at least once, exactly once
* We want exactly once

### Data deduplication

* Can come from clients trying to click the same ad multiple times
* Server outage 
	* If node goes down and upstream doesn't receive an ACK, it may try again

![[Pasted image 20240213092215.png]]

Use HDFS or S3 to record offsets before the crash

If it crashes before now, there could be message loss because the offset never updated
* Need to save the offset exactly onces the downstream kafka gives an ACK

It's not easy to dedupe data in large-scale systems. Exactly once is quite difficult

#### Scale the System

Adding more consumers can take a few minutes to rebalance, be careful and do it during off-peak hours

Brokers
* Hasing key
* Number of paritions
* Topic physical sharding
	* Partition by Country
##### Scale aggregation service
1 - Allocate events with differnt ad_ids to different threads like figure 6.23
2 - Deploy it on something like Apache Hadoop or YARN (Resource providers)
![[Pasted image 20240213092739.png]]
Option 2 is more widely used, but option 1 should be easier to implement
##### Database

Cassandra natively suppports horizontal scaling


### Hotspot issue

Major companies can place millions in ads. Therefore they will get more clicks
If someone is popular, we can add more aggregation nodes to them
* Use resource manager to apply for extra

![[Pasted image 20240213093021.png]]

#### Fault Tolerance

Aggregation service is in memory, so it can go down and all data is lost.

Best to save an offset and redo from there

Take snapshots

#### Data monitoring and Correctness

Critical to monitor the system's health to ensture correctness because we deal with billing and RTB auctions

**Continuous Monitoring**

Measure Latency and Message Queue Size
* If it rapidly gets big, then we need to add more aggregation nodes
* Long latency can also show signs of bad things
* System Resources like CPU, disk, JVM

#### Reconcilation

Don't have bank records that we can check.

We can sort the ad data though for every partition at end of day and compare with real-time data in the past hour?



![[Pasted image 20240213093435.png]]

Generalist System Design interviews don't require internals of different pieces of specialized software used in a big data pipeline

Discussing thought process and trade-offs is very important.

This chapter focused on the best results for a generalist system design interview


---

Stream
* Low Latency, High Throughput
* No bound on amount of data that comes in
Batch
* High limit on data that comes in
* High Throughput*
System???? OLTP?
* Has high availablity*

How to Aggregate the in 24 hours or whatever
* Use a sliding window alongside a stream of data
* Tumbling Window -> Parition time into slices and then aggregate on that

Scale Consumers by adding / removing nodes based on the flow of data
* Add consumers during off-peak hours to minimize downtime / effects

Resource Manager on the nodes to balance loads?
Threads to balance loads?




