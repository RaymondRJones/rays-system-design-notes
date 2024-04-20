It's like Yelp / Google Maps
## Clarify questions 
* Will the range expand if nothing is available?
* Will businesses sign themselves up?
* What happens if the user is moving?
##### Back of Envelope Estimations for 100M Daily Users and 200M Businesses
* 100k (10^6) seconds in a day
* 1 (10x8) / 10^5 -> 10^3 -> 1,000 QPS
	* 100M
		* 100(10x6)
			* 1(10x8)
* users make 5 queries every day
	* 5 x 1,000 QPS -> 5,000
* 5,000 QPS

Obvious heavy read application with low writes
	We'll have an API that has a GET for search/nearby
		And API that handles CRUD for add,update,delete businesses
	GET -> /v1/business
	POST -> /v1/business
	PUT -> /v1/business
	
How do we split the earth to get all of these businsesses
* Naive -> Search 2-D array space of Lat / Long
* Zip-codes / Cities / Even distribution
* Geohashing -> Split earth into Lat/ Long by recursively going downward
	* 00010100
* QuadTree / R Trees
* Google S2 Geometry Library

#### Step 3 - Deep Dive

Scale the DB
* Business Table -> Can use sharding on business ID???
* Geospatial Index Table
	* Better to add double rows for businesses in the same geohash
	* Option 1 is to add multiple columns for a geo hash of a business (This is slow for updating, add, and delete)
* Scaling Geospatial isn't necessary because it's only at 1.71GB of memory
* Depending on QPS, the server needs mroe CPU or network bandwith
	* Spread readload over mutliple database servers
* Xu says it's easier to just use read replicas, sharding gets too complicated with geohash

#### Geohashing vs Quadtree

Geohashing is easy to update and easy to implement

Quadtree requires traversing tree to delete nodes
* Also more complicated to implement
* And need to build the tree 


Cache
* Not needed, dataset is so small and works within server memory
* Data retrieval is just as fast as in memory
* Need to show numbers during an interview??
* Use the geocache NOT the long/lat. Long/lat changes so often
	* Use Redis
* 200M business --> 4 radiii -> 
	* Each business is one grid
	* about 5GB of memory at 8KB per 200M businesses and 3 precisions
* Can cache business ID for their pages to show to client

Region and Availability Zones
* Make sure to route users at data centers that are near them
* Keep track of privacy and how data is stored

Followup -> How to filter by time

Can filter post because we'll only have like 100 businesses that we show near them
* assuming open time and closing time is inside the table


