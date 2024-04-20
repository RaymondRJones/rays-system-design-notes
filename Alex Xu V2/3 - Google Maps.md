Helps users navigate to their destinations
* has review platform too
* Can see alternative routes

# Step 1 - Clarify and Scope Problem

How many users do we expect?
* 1 Billion DAU
Which features should we focus on?
* Navigation? Direction? ETA?
	* Focus on location update, navigation, ETA, and map rendering
How large is the road data? Do we have access to it?
* We do have access, and it's TBs of raw data


Should our system take traffic conditions into consideration?
* Yes, traffic conditions are important

How about different travel modes? Biking, walking, etc.
* Support different travel modes

Should it support multi-stop directions?
* That can be a follow up

How about business places and photos? How many photos are we expecting?
* I'm happy you asked and considered these. We do not need to design those

## Functional Requirements

Update user location
Navigation Service -> ETA Service
Map Rendering
### Non-Functional Requirements

Make sure it's available and scalable
Okay for there to be slight delays in data
Accuracy -> Users can't be given the wrong directions
Data and battery usage -> Should be minimized
Smooth map rendering

#### Useful Concepts for Google Maps

Map 101
* Positioning System
	* Latitude and Longitude
* 3D to 2D
	* Map projection
		* Several ways, most of them distort the true map 
* Geocoding
	* Convert addresses to geographic coordinates
	* So "1600 A... Parkways" becomes (37.123, -122.0121)
* Geohashing
	* Encodes geographic area into a short string of letters and digits
	* string of numbers between 0 and 3 made recursively
	* Tiny amounts of 4 subgrids
* Map Rendering
	* Tiling -> User only downloads the area of the map that they need to see
		* There are distinct sets of tiles at different zoom levles
* Road Data Processing for Navigation Algorithms
	* Dijktra's or A* pathfinding variations
	* Don't worry about when to use which
	* Important part is the idea of graphs
	* Can't have entire world of roads in a single graph
	* Break up graphs
		* Similar to tiling
		* Routing Tiles Creation
	* Hierarchical routing tiles
		* Routing levels change depending on the size
		* If we want to travel from US to Colombia, we can't use a massive graph
			* Strategy must change
			* Only contains local roads vs. only contain interstate roads

### Back of Envelope Estimation

Need to store map of world
Metadata for each map tile -> Negligible for each map tile
All roads in world (TBs)

Map of world
* Final zoom will be 256 x 256 PNG image at a size of 100KB
	* Zoom level 21 -> This has 4.3 trillion tiles
	* 4.3T * 100 KB = 440 PB
* 80 to 90% of the Earth is just ocean, desert, etc.
	* High compression for these
	* 44 to 88PB
	* Let's say 50PB to be clean
* Lower zooms will reduce the number of images and tiles
	* drops by 4x every time
	* 50 / 4 + 50 / 16 ... = ~67PB


### Server Throughput / Load

Two types of requests
* Navigation
* Location Updates

1 Billion DAU
Let's say each user uses 30 minutes per week of navigation
35B minutes per week or 5B minutes per day
Send location updates every second?
* 5B minutes per day * 60 -> 300B
* 300B / 10,000 = 3 million QPS
Every client may not need every second, We can batch this on the client
* Reduces write QPS*
* If we do it every 15 seconds we now get
* 3 million / 15 = 200,000
* QPS can be reduced to 200,000

*What does he mean by Batch requests? Combine them?*

Assume QPS peak is 5x th eaverage -> 200,000 x 5 = 1 million

# Step 2 - Propose High Level Design and Get Buy-In


![[Screenshot 2024-02-10 at 10.49.50 AM.png]]


We have a ton of writes, so the database should be something with high-write ability
* Cassandra
Even with batching, we have high writes
* Batching is the idea of waiting ot send the locations to the servers
* Wait an extra 14 seconds to reduce load

Can log location data using Kafka too for further processing

HTTP works for a communication protocol. It's straightforward and has "keep-alive"
* no idea what keep alive is
* can look like `POST /v1/locations`
	* Parameters -> locs: JSON array of (Lat,long,timestamp)


#### Navigation Service

Finding shortest / fastest path from A to B
* Adaptive / ETA service to calculate speed
* This service will only handle shortest path for now

## Map Rendering
* Map tiles are 100 PB in size
	* Can't hold this on the client
	* Need to fetch from the server
* If the users zoom to a location, we should fetch
* If the user's navigation point moves to a new area

Option 1
* Server builds map tiles based on the clients location and zoom level
* Puts huge load on the server to generate every map tile each time

Option 2
* Fetch map tile using geohash of where it should be
* Use a CDN for static pages
* Can be much faster for fetching, find a datacenter near the user

Data-usage of user estimation
* 30km/h for 100KB initial image and zoom level, we'd need 25 images or 2.5MB of data for a 1km x 1km area
* At 30km/hour we'd need 75MB of data per hour or 1.25BM of data per minute

CDN usage
* 5 billion minutes of navigation a day
* 5 billion * 1.25MB* is 6.25B MB per day
* Serving 6.25B / 100,000 seconds in a day
	* 62,500 MB per second
* Served from POPs along the world, each POP will serve about 
	* 62,500 MB / 200 per second

Can have the client geohash their location and then send to our servers to retrieve their needed map geohash
* Interesting tradeoff is do to this on their client's device
	* Bad because if we change our scheme, we also need to change our device code
* Or do this on our server
	* Have more flexibility if we change our schema


## Step 3 - Design Deep Dive

### Data Models

Routing Tiles

* We're given list of roads with info like, county, lat, long, city name
* Need to pre-process it into a graph
	* Run Periodic offline to transform dataset into routing tiles / graph
	* Runs periodically to see any new differences
* Graph is too big to store in memory
	* Graph DB?
	* We don't need any DB features so we're wasting CPU power
	* He proposes AWS S3 buckets and cache it aggressively
	* Organize the files by their geohashses for fast look up time

User Locations
* user_id, lat, long, timestamp
* He also adds an "active" and "is_driving" mode
Geocoding Data
* KV store
Map Tiles of the World
* Store on a CDN backed by AWS S3 Bucket, use Geohash

### ETA Service

Get a list of shortest possible paths

Use machine learning ot predict ETA based on current traffic and historical data
 
 Rank these outputs, filter them (No tolls, etc.)


 