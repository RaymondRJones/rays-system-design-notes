#### TLDR
* Intro to Distributed Systems
	* Multiple servers (brokers)
		* Meta data storage
		* Discovery Services (ZooKeeper)
	* Replication and Leader / Follower
	* Pub / Sub Model
	* Point to Consumer
* WAL as a Database
	* Batch writes a needed here
		* More throughput, lower latency


# The Problem

Message queue are the building blocks between systems
* Decoupling
	* Reduce tight coupling between components
* Improved scalability
* Increased availability
	* If a system goes offline, messages can still be held
	* Other components can talk to the queue
* Better performance using async responses
	* Producers don't need to wait

Most popular Message Queues
* Kafka, RabbitMQ, Apache RocketMQ, Apache ActiveMQ

## Message Queue vs. Event Streaming Platforms

Kafka and Pulsar are technically event streaming platforms

Event streaming platforms have more features
* Allow repeated message consumption, long message retention. Append-only log
* This chapter will involve a distributed message queue with additional features like the line above

### Step 1 - Understand the Problem and Establish Design Scope

Producers send messages, consumers consume them
What about the performance?
* Message delivery semantics
* Data detention

What's the format of the messages and their size? Is it text only? Is multi-media allowed?
* Text messages in the range of KBs

Can messages be repeatedly consumed?
* Messages can be repeatedly consumed (This is an added feature, not traditional MQ)

Are messages consumed in same order they were produced?
* Yes
Does data need to be persisted and what is the data retention?
* Yes, data retention is two weeks (This is an added feature)

How many producers and consumers will we support
* The more the better

What data delivery semantic do we support? At-most once? at-least-once? Exactly once?
* Support at-least-once, but ideally we could support all of them

What's the target throughput and end-to-end latency?
* It should support high throughput for use cases like log aggregation. It should also support low latency delivery for more traditional message queue use cases.

#### Functional Requirements
* Producers send messages to queue
* Queue stores messages for 2 weeks
* Consumers consume messages from queue
* Messages can be consumed repeatedly or only once
* Message size in KBs
* Deliver messages to consumers in order they were added
* Data delivery can be configured

#### Non-Functional Requirements
* High Throughput or Low Latency, configurable based on use cases
* Scalable. The system should be distributed in nature. It should be able to support a sudden surge in message volume
* Persistent and durable 

# Step 2 -> Propose High Level Design and Get Buy-In

#### Messaging Models

Point-to-point
* Message only consumed by a single consumer
* Consumer acknowledge that it consumed it, and it's removed
* Locks? for multiple consumers?

Publish - Subscribe
* Topic
	* Categories to organize messages
* Messages are sent to topics and received by consumers who subscribe to the topic


consumer group for point to point

pub-sub for topics


#### Topics, partitions, and brokers
* How do we scale?
	* What if data volume is too large for a single server?
	* Partitions and sharding

Message is sent to one of the partitions
* Can organize by key (user_id), or randomly do it
* Each consumer has it's own partition to scan for

#### HLD

![[Screenshot 2024-02-11 at 10.49.11 AM.png]]

Client
* Producer pushes messages
* Consumer subscribes and consumes messages

Core service and Storage
* Broker -> ServeHolds multiple partitions
	* Servers that hold the partitions are called brokers
* Storage
	* Data storage -> Messages are persisted in data storage partitions
	* State Storage -> Consumer states
	* Metadata storage -> Configuration and properties of topics

Coordination service
* Service discovery
* Leader election -> One of brokers as an active controller to assign partitions
* Apache ZooKeeper or ETCD to elect a controller

# Design Deep Dive

He chooses to use an on-disk data structure that takes advantage of great sequential access performance of rotational disks and the aggressive disk caching strategy of modern OS

Messages that are passed are not modified to reduce copying

His design encourages batching. Producers send messages in batches. Message queue persists message in even larger batches. Consumers fetch in batches if possible

### Data storage

Write-heavy and read-heavy

No update or delete operations

Predominantly sequential read/write access

##### Option 1: Database

Relational Database
* Create topic tables and store messages as the rows
No SQL 
* Collection as topic, message as documents

Database doesn't support write-heavy AND read-heavy

#### Option 2: Write-ahead-log (WAL)

A plain file where new entires are appended
* Use as redo log in MySQL and the WAL in ZooKeeper

LOOK THIS PART UP [5]

Disk performance of sequential access is very good. Rotational disks have large capacity and are pretty afforable

Disks are slow for random access, but we can get track of our needed data
* Easily allows several hundred MB/sec of read and write speed

#### Message Data Structure

Field | Datatype

**key | byte**
**val | byte**
topic | string
parition | int
offset
timestamp
size
crc | int


Message key is used to determine the partition of the message, if it's not defined thenw e do random
* value is a payload of the message

The message can be found using partition, offset, and topic

### Producer Flow

Which broker / partition should the producer sent the message to?

We need to service that routes the messages to the proper broker / parition

Producer sends message to routing layer
Routing layer reads # of partitions available from metadata storage
Routes message to the active leader to decide
Followers copy the data too
Leader commits data and sends and "ACK" to the produer

To improve availability this design also uses replication
* Improves Fault Tolerance

###### Drawbacks

New routing layer means more network latency
Request batching isn't considered here

###### alternative

We add the routing layer into the producers and consumers directly

#### Consumer Flow

##### Push vs. Pull Model

Push model
* low latency, message read as soon as it arrives
* cons ->
	* Too many messages could arrive and consumers are overwehelmed
	* Brokers control the rate, so hard to deal with powerful / diverse consumers

Pull Model
* consumers control the rate
* Can add more or less consumers to fix the rate
* Better for batch processing
### Replication

ISR -> Checks which replicas are in sync with the leader
* Not in sync means they cannot be used
	* Set the number of messages that you want before it can't be used anymore

ACK