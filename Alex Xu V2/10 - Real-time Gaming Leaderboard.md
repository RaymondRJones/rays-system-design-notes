# TDLR
* Skip lists are a thing
* SQL databases have a limit
* Making a basic API
* Sorted Data can help queries


# The Problem
Leaderboards are common in gaming and elsewhere to show who is leading a particular tournament
# Step 1 - Understand the Problem and Establish Design Scope

How is score calculated for leaderboard?
* Gets a point for winning a match

Are all players on the leaderboard?
* Yes
Is there a time segment associated with the leaderboard?
* Each month, a new tournament kicks off which starts a new leaderboard

Do we only care about the top 10 users?
* Yes, we can do other things later

How many users are there?
* 5M DAU and 25M monthly active users

How many matches are palyed during a tournament
* 10 matches per day

How do we determine the rank if two players have the same score?
* They have same rank

Is leaderboard real-time?
* As close to real time as possible

### Functional Requirements

Display top 10 players on the leaderboard
Show user's specific rank
Display players who a four places above and below a desired user (Bonus)

### Non Functional Requirements

Real time updates
Score update in real time on leaderboard
General scalability, availability and reliability requirements


#### Back of Envelope Calculations
There's likely going to be peak hours for gameplay, not all over 24 hours

5M DAU x 10 games? -> 50 x 10 ^ 6 / 10 ^ 5 -> 500 QPS 
* 50 QPS for people checking the leaderboards (once per day)
Storage
* 25M MAU x 2KB
	* 50 10 ^ 6 x 10^3 -> 50 x 10^9 -> 50TB per month

# High Level Design and Get Buy-In

/v1/scores -> Add new score
* user_id
* points
* Returns
	* 200 OK
	* 400 BAD Request

/v1/scores -> Get top 10 players
Sample Response:

![[Pasted image 20240217081914.png]]

GET /v1/scores/{:user_id}

get rank of specific user

### High Level Architecture

![[Pasted image 20240217082204.png]]

Should client talk to leaderboard service directly instead of going through game service?
* JS can be manipulated, so probably not

Do we need a message queue?
* Useful if we need to send the data to serveral services
* If the data is going to be used in other places or support other functionality

### Data Models

#### SQL

If scale is small we can do SQL
* 500 QPS is honestly pretty low, I think SQL works here
![[Pasted image 20240217082447.png]]

Relational database can't handle the number of reads that we need
* 50 QPS

Relational can't work with millions of rows
* To find the top 10, we'd have to constantly sort millions of rows

We could use batch operations, but then it's not a real-time leaderboard

#### Redis Solution

Key, Value store
Also has `sorted Set`
* Like a set data type
* Can sort the data based off their values, like `score`
* Hash table with a skip list
	* Faster searching than a linked list

### Implementation using Redis Sorted Sets

ZADD -> Inserts or Updates for O(logN)

ZINCRBY -> increment score, assume it starts at 0. O(logN)

ZRANGE / ZREVRANGE -> fetch range of users (logN + M)

ZRANK -> Fetch position of user in ascending / descendin in logarithmic time

## Workflow within sorted sets

![[Pasted image 20240218082734.png]]

User fetches top 10 ranks
![[Pasted image 20240218082821.png]]

![[Pasted image 20240218082840.png]]

### Storage Requirement

25 MAU storing 26 bytes is ->>> 25 x 10^6 x 26 -> 650 MB

More than enough for a single redis server

2500 QPS is easily handled by Redis

Persistence
* Node can fail
	* Redis has persisteance though
* Restarting Redis from a disk is slow
	* Usually configured with a read replica, but when main is fails, the read becomes the leader
	* New read is set up

Use a SQL table along with the Redis so we can story match history and other things

# Step 3 -> Design Deep Dive
#### Cloud Provider vs. Own Servers

![[Pasted image 20240218083547.png]]

Using AWS Lambda + AWS Gateway

![[Pasted image 20240218083719.png]]

Benefit is that it autoscales. Xu recommends the Lambda way instead of building own servers like AWS EC2
* Granted you still need servers for the Redis and MySQL
* Pretty sure AWS doesn't have one of those? Maybe they do though like DynamoDB
### Scaling Redis
* Xu mentions Sharding to expand our Redis

#### Fixed vs. Hash Partitions

Partition by the number of points won, from 1 to 100, 100 to 200, etc.
Query the final partition to get the winners

Hash users and put them into random buckets
* Querying for top 10 is more difficult
* Need to search every partition


### Alternative solution: NoSQL

Optimized for writes

Efficiently sorting items

![[Pasted image 20240218084414.png]]

Ah and now he mentions DynamoDB

Stuff about a "global secondary index" that's apparently completely made up from a non-technical AWS Salesperson


#### Wrapping up

System failure recovery
* Record timestamps in MySQL and then read through all the records
	* Increment every time the user won game on the new redis server


