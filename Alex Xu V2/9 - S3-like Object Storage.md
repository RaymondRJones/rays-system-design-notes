S3 Launched in 2006
* Versioning, bucket policy, and multi-part upload support in 2010
* Server side encryption, multi-object delete, object expiration in 2011
* 2 trillion objects stored in 2013
* Life Cycle policy, event notification, cross-region replication
* 100 trillion objects storied in S3 in 2021

### Storage System 101

There's three broad categories of storage

Block Storage
File Storage
Object Storage

##### Block Storage
* Came out in 1960s
	* SSD and HDD are attached to servers
	* Most versatile because applications can use the blocks / volumes of storage, or they can use the file system of the OS

File Storage
* Built on top of block storage
* Data stored as hierarchial structure
* SMB/CIFS and NFS are network protocols to transfer and share storage between servers
* Has great simplicity to share files within an organizations

Object Storage
* New, sacrifices performance for durability
* Stores objects in a flat structure, no hierarchy
* Data accessed via RESTful APi

![[Pasted image 20240216082313.png]]

![[Pasted image 20240216082328.png]]

#### Terminology

Bucket -> Container for objects

Object -> Individual piece of data we store in bucket

Versioning -> Multiple variants of an object in the same bucket

Uniform Resource Identifier (URI) -> Object storage provides RESTful APIs

Service - level agreements (SLA) -> Contract between service provider and client
* Design for durability of 99.9999999% of objects on multiple availability zones


# Step 1 - Understand Problem and Establish Scope

Which features should be included in the design?
* Bucket creation
* Object uploading and downloading
* Object versioning
* Listing objects in a bucket, like AWS s3, `ls` command

What's typical data size?
* Few GBs or more and large number of small objects (tens of KBs) efficiently

How much data do we need to store in one year
* 100 PB

Can we assume durability is 6 nines? and service availability is 4 nines?
* *(Note I like this question for showing knowledge of nines*)
* Sounds reasonable

#### Non-functional Requirements
* 100PB of data
* Data durability of 6 nines
* Service availability of 4 nines
* Storage efficiency

### Back of Envelope Estimations
Object storage has bottlenecks of disk capacity or disk IO per second

Disk Capacity
* Let's assume
* 20% are small
* 60% of objects are medium
* 20% are large objects

IOPS -> one hard disk can do 100 - 150 random seeks per second (100 - 150 IOPS)

Use median of small, medium, and large object to simplify math
40% usage ratio gives

100PB = 100 x 1000 x 1000 x 1000 MB = 10^11 MB
10^11 x .4 / (.2x.5MB ... .... ..) = .68B objects

Probably won't use these numbers but Xu likes to have numbers

## Step 2 -> Propose High level design and get buy-in

Characteristics of Object Storage
* Object immutability
* Key-Value Store
* Write once, read many times
* support both small and large objects

File storage in UNIX works by writing inodes that point to where the data is located

![[Pasted image 20240216085007.png]]

Load balancer

API service

Identity and acces management 

Data store

Metadata store

#### Uploading an object
![[Pasted image 20240216090749.png]]
### Downloading an Object

![[Pasted image 20240216090805.png]]

# Step 3 - Design Deep Dive
### Data Store

Has three main components

![[Pasted image 20240216092328.png]]

Data Routing uses RESTful or GRPC APIs to access datanode cluster. It can scale by adding mroe servicers
* It queries the placement service to get best data node to store data
* Reads data from data nodes to give to API service
* Writes data to the data nodes

###### Placement Service
* Uses a virtual cluster map of all data nodes
* Uses heart beats to make sure the data node is available
* Also contains location info for each data node to make sure replicas are physically separated
* Paxos or Raft consensus protocol can build this service, best to have 5 or 7

###### Data Node

Stores the actual data
Has a data service darmon running on it
Sends heartbeats to the placement service
* Says how much data is stored on each drive
* Says how many disk drives the data node is managing

When placement receives the first ever message, it sets it on the virtual cluster map and gives a unique ID

### Persisting Node Data
![[Pasted image 20240216092856.png]]

Notice the replication

Primary node responds once writes complete and the data routing service ACKS and sends object ID back to API

Can choose which level of replication you want. There's a tradeoff between consistency and latency
* ACK after 2 successful replicas (Slowest, but safest and highest consistency)
* ACK after 1
* ACK after 0 successful replicas but a success primary (lowest latency)

#### How to store data

I'm skipping this section

Goes into details on how file storage works

Need the start offset of the object along with it's size

### Durability

Hardware failures and failure domain
* We replicate data 3 times because each hard drive has about a 0.81% failure rate per year
* 3 data copies gives 0.999999 reliability or something

Failure domain requires isolating the physical environment, so one environment going down doesn't destroy everything
* Rack-level domain -> Everything for a server rack if it goes down

This is why Xu recommends availability zones
![[Pasted image 20240216093614.png]]

##### Erasure Coding to boost durability

Data is spliced into smaller pieces and places on different servers
Creates parity
If a failure occurs, we can reconstruct the pieces

![[Pasted image 20240216093742.png]]

You don't lose 100% of the data

![[Pasted image 20240216093814.png]]
Erasure has cost efficiency, don't need to spend money on more stoarge

Greatly complicates node design though

#### Correctness Verification -> Data Corruption

In-memory data corruption happen a lot in big systems

Use checksums to verify if the data was correct
* looks like a hashing algorithm that you can run to see if the data is the same

MD5, SHA1, HMAC

#### Metadata data model

Find object ID by name
Insert and delete an object based on object name
List objects in a bucket sharing the same prefix

![[Pasted image 20240216094536.png]]

#### Scaling the bucket

There's a limit on buckets a user can create so the bucket size is small. So scale isn't a problem

#### Scaling the object table

Holds the metadata for the object. 

Can scale this table with sharding.
* Which key to use?
* bucket_Name can introduce hot key issue
* object_id works but makes key range queries hard for query 2 and 3
	* Maybe normalize?
	* Use a hash of the bucket_name and object name as sharding key


### Closing out this chapter to maybe do later
Things mentioned..

Query to list objects in bucket
Single vs. Distributed databases for this query
Uploading large objects
Garbage collections
object versioning



