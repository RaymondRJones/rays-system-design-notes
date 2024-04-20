# TLDR
* Servers and Datapools to manage 
	* Can use Zookeeper to Coordinate
* Time series databases are a thing
	* They even have their own cache query
	* Can handle like 200,000 QPS for writes
	* 15 QPS for reads
* Kafka and Message Queue help scale data transmission
* Can push server data or pull server data
	* have their pros and cons
	* Pushing means server needs Zookeeper to not be overloaded
	* Pulling means we can't aggregate metrics on the server level
* Decide when to aggregate the data
* Decide how to expire data
	* Cold storage
# Problem
Popular services include
* Datadog
* Influxdb
* New Relic
* Grafana
# Step 1 - Understand Problem and Establish Scope

Who are we building the system for? Is it in-house? Or is it SaaS?
* Internal use only

Which metrics do we want to collect?
* Operational system metrics
* CPU load, memory usage, disk space consumption
* Also include requests per second or running server count

What is the scale of the infrastructure we are monitoring with the system?
* 100M DAU, 1,000 server pools, 100 machines per pool
* 1 x 10^9 / 10^5 --> 10^4 QPS -> 10,000
* 10 ^ 2 x 10^3 -> 10^5

How long to keep data?
* 1 year

Do we reduce the resolution of the mtrics data for long-term storage?
* Keep newly received data for 7 days. 
* Roll up to 1 minute resolution for 30 days
* After 30 days, 1 hour resolution

What are supported alert channels?
* Email, phone, PagerDuty, webhooks

Do we need to collect logs?
* Error log or access log?
* No

Do we need to support distributed system tracing?
* No

### High-level requirements and assumptions

Now you have finished gathering requirements fromt he interviewer and have a clear scope of the design. The requirements are:
* The infrastructure being monitored is large-scale
	* 1,000 sever pools, 100 machines per pool, 100 metrics per machine
		* 10 M metrics
	* 1 year data retention
		* reduces after 7, 30 days
	* CPU, Request count, Memory, message count in querries

## Non-Functional Requirements

Scalability
Low Latency
Reliability
Flexibility

Out of scope
* Log monitoring (Kibana / ElasticSearch)
* Distributed System Tracing

## Step 2 - Propose High-level Design and Get Buy-in

#### Fundamentals
* Data Collection
* Data Transmission
* Data Storage
* Alerting -> Analyze incoming data, detect anomanlies, generate alerts
* Visualization -> Charts / Graphs / etc.

#### Examples of Metrics 

1 -> Finding CPU Load of a specific machine

Time series Object
* Metric name
* Set of tags/labels (KV pairs)
* Array of values and their timestamps
	* Value/Timestamp pairs

### Data Access Pattern

Very write heavy, there's a lot of time data points

10M operational metrics written per day

Read load is bursty / spiky. Can have data bases that really want to see the metrics for alerts or visualizations

Data Structure is too hard to use SQL for

Cassandra is great for heavy writes though
* Requires heavy knowledge of NoSQL to devise a scalable schema for querying time series data

There are databases designed for timeseries data
* OpenTSDB
* MetricsDB and Amazon offers Timestream
* InfluxDB and Prometheus

You don't need to know the internals of these databases because it's so niche
* Unless you mentioned that you know it on your resume
![[Screenshot 2024-02-12 at 8.21.38 AM.png]]
# Step 3 - Design Deep Dive

### Metrics Collection

Dataloss isn't terrible for counters or CPU Usage

Pull vs. Push models

Pull
* Metrics Collects waits every X time to pull
	* Collector needs name of every machine / service endpoint
	* Hard to maintain at large scale with so many machine updates and additions
	* It could talk to ZooKeeper which can give a list of all server nodes
* Also need multiple pools of metric collectors, won't work with one server at this scale
	* Need coordination so each one doesn't pull from same server
* Use consistent hashing to assign each one

Push Model
* Push collector is installed on every server, and it routinely sends data to metrics collectors
* Can also aggregate the data before sending it
* Should install a load balancer to a metrics collect doesn't get overloaded and data loss occurs

Pros and Cons to both
![[Pasted image 20240228083211.png]]

### Metrics Transmission

Probably use Kafka for scale
* Will cause delays, but make sure no data is lost
* He says Apache Storm, Flink, and Spark and act as consumers

We can further do partitions through Kafka
* Configure number of partitions
* Parition based on metrics data like names
* Categorize and prioritize certain metrics first

Alternative to Kafka
* Facebook Gorilla Database?
* Interview may ask what other alternatives are there

### Where to Aggregate the data?

Client-side can only aggregate simple things

Ingesion Pipeline
* Need a stream processing like Flink if we want to aggregate before writing to a DB
* Lose flexibility because we lose raw data, but we drastically reduce writes

Query side
* Query speeds are slower because there's more raw data

Query Service
* Can have several servers dedicated to querying the DB and handling requests to aggregate

Cache Layer
* Store query results

Most popular industrial scale services already have a query service and cache layer built-in or through plug ins

Flux has it's own built-in query language, like other metric queries
* AWS Cloudwatch has this too
### Storage Layer

Compression

Downsampling
* 10 second data becomes 30 second data

Cold Storage
* Inactive data that's rarely used
* Use thirdpart visualization and alerting systems instead of building our own

## Buy or Build your own Alert / Visualization

Grafana is very good








