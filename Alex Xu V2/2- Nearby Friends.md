#### Clarifying questions I have before seeing the problem 

What's the max radius that friends can see other friends?
Should it notify them if a friend is nearby?
The user can open the app and see their friends?

Can people send friend requests to other people?
* And remove friends?
Do they have profiles or photos?

Can they see the exact location of their friends?
### Step 1 -- Clarify

How many users does the app have?
* We assume 1 billion users and 10% use nearby friends feature


Can we assume distance is calculated in a straight line between two users? Like if there's a river, that could technically add travel distance time.

Do we need to store location history?
* Protect user location data?

Inactive friends for 10 minutes will disappear? Or do we display last known locations?
* Remove inactive friends

## Step 2 - Requirements

##### Functional Requirements
* Users should see list of nearby friends on mobile app, each entry has a distance and timestamp when distance was last updated
* Nearby friend lists should be updated every few seconds

### Non-Functional Requirements

Low latency -
Reliable - Relaible overall but Occasional data point loss is okay
Eventual Consistency (Not Strong) - Seconds delay from receiving location data from replicas is okay

### Back of envelope estimations

1 B users
10% use the feature
100M users for this feature
* ~~Let's say 100M people check it every day?~~
* 10% of users are DAU
* 10M
* Average user has 400 friends
* Update their location every 30 seconds
Location Update QPS = 10M / 30 -> 334,000

~~100M * Query = 100M Q per day
100,000 seconds in a day 
100,000,000 / 100,000 -> 100
100 Queries per second~~

#### Storage

100M people's locations
Long / Lat -> 8 Bytes
2,000 different locations on average?

100M * 8 * 200? -> 800M bytes * 2,000

# Step 2 -> High Level Design

Problem requires efficient message passing
* I thought of doing fan-out
* Could be done peer-to-peer -> Every user has an active connection to other friends
	* Not great with mobile, connection can flake. Power consumption si higher

![[Screenshot 2024-02-09 at 7.20.48 AM.png]]

Receive Updates
Find all active friends and fan out
Is distance for friend is too high, don't fan out to them

#### Cons - Doesn't work at high volume

10M users that are updated every 30 seconds
334k QPS
At 400 friends (10% of them active) that's 14M Updates per second
14M writes to a database every second is a lot

# Proposed Design

Mobile users use web socket and http

Load balancer 

Send into Web Socket Servers (Bidirectional location info)
* Redis Pub/Sub, Location Cache, Location History DB, User DB
* Fanout on a Pub/Sub is optimzied?
* TTL on the cache means that we can declare users as inactive
* User Database stores which users are friends AND user data
	* Can be Relational or NoSQL
* Location history database -> Historical location data
* GBs of memory Redist Pub/Sub can hold millions of channels
	* Friends subscribe to a channel, users push to their channel
	* Why are web sockets needed for this though??
* 




Send to API servers (Auth, User Management, Friend Management)

![[Screenshot 2024-02-09 at 7.36.09 AM.png]]

#### How well does each component scale?

##### API Servers
* Just add more servers, CPU, RAM, etc.
* Other ways to auto-scale too
* Kubernetes???
##### WebSocket servers
* Striaghtforward to autoscale
* Becareful when deleting nodes though
	* Have to "drain"
	* Mark it as "draining" in load balancer so things aren't sent to it
* Stateful server autoscaling is the job of a good loud balancer
##### Client Initialization
1. Update location in cache
2. Save location for WS
3. load all user's friends from database based on user_id
4. Makes batched request to the location to fetch X friends
5. Computes distance and see if it's at the maximum
6. Send to the client if it's good
7. Subscribes user_id to the friend's channel
8. Sends current location to user's channel in Redis Pub/Sub server
##### User Database
* User_id, name, profile URL, friends
* Won't fit for 10M in a single database
* Can use sharding

Location Cache
* will hit 334k writes / updates per second
* Likely too high for redis, can use sharding on user_id's to improve performance
* To improve availability, add replicas

#### How many Redis Pub/Sub Servers do we need?

At 100M users, each with 20B for 100 friends
* 200M Bytes * 100 friends
* 20B Bytes
* 200 GB

CPU Load
* 14M writes per second
* Let's say the max load is about 100k writes per second
* We need to shard 14M / 100,000
	* 140 Redis Servers
* Where should these be located?
	* Service Discovery with Zookeeper

How do we Autoscale these servers
* Will need to create a new hash ring as we grow
* This is like a stateful scaling
* Downsizing means losing state data





