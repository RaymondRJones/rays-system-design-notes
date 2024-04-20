
Similar to AirBnB, Flight Reservation System, and Movie Ticket Booking System

# Step 1 - Understand the problem and establish scope

What's the scale of the system?

Can customers pay when they arrive to the hotel?
* No

Do customers only book through the website?
* Yes, app and website

Can they cancel?
* yes

What other things should we consider?
* Allow for 10% overbooking and prices change dynamically
So a list of features looks like
* Show hotel-related page
* Show hotel room detail page
* Reserve a room
* Admin panel to add/remove/update info
* Support the overbooking feature
#### Non functional requirements

Support high concurrency -> During peak season, people booking at same time can occur

Moderate latency -> Fast response time when user makes a reservation, like a few seconds before they get an ACK from the server


### Back of the envelope estimation

5,000 hotels. 1 Million rooms in total

70% of rooms are always filled and average stay duration of 3 days

1M * .7 / 3 days -> 233,333 queries/day

QPS -> 2.33/second

##### QPS of every page in the system

View hotel page ->
View booking page ->
Reserve a room ->

Let's assume a 90% loss of users at each step

![[Pasted image 20240213184506.png]]

### Propose High Level Design and Get Buy-in

#### API Design

###### Hotel Related APIs

GET /v1/hotels/ID -> Get info about a hotel
POST /v1/hotels -> Add new hotel (only for hotel staff)
PUT /v1/hotels/ID -> Update hotel info
DELETE /v1/hotels/ID -> Delete a hotel

##### Room related APIs

GET /v1/hotels/ID/rooms/ID -> Get info about a room
POST /v1/hotels/ID/rooms -> Add new room (only for hotel staff)
PUT /v1/hotels/ID/rooms/ID -> Update room info
DELETE /v1/hotels/ID/rooms/ID -> Delete a room


##### Reservation API

GET /v1/reservations -> Get reservation history for logged in user (some session id?)
POST  /v1/reservations/ID -> Add new room (only for hotel staff)
PUT /v1/hotels/ID/rooms/ID -> Update room info
DELETE /v1/hotels/ID/rooms/ID -> Delete a room

#### Data Models

First let's look at each query to get a sense of volume 

Query 1 -> View hotel info
Query 2 -> Find rooms based on date range
Query 3 -> Record a Reservation
Query 4 -> Look up past reservation or history of reservations

He decides to use a relational database because this is not a write-heavy application
* Work well with ready-heavy and write less frequently workflow
* NoSQL databases are generally optimized for writes (Although isn't there DynamoDB?)
* SQL guarantees ACID, this is huge for the reservation system
* It's easy to model the data

![[Pasted image 20240214072152.png]]


Status is the only weird one here. We do it for locking, recording cancelled, and checking if we've refunded yet

##### High Level Design

![[Pasted image 20240214073749.png]]

CDN to cache the static assets like the HTML pages, images, videos, JS.

Internal APIs -> Only available to hotel staff
* Can requiere a VPN and allow allow access with select internal software or websites

## Step 3 - Design Deep Dive

### Improved Data Model

For a hotel, we use room types, so we need to change the names to room types in our schema

![[Pasted image 20240214082310.png]]

The Reservation service changed significantly though

It has an inventory system that counts the number used and available

Xu doesn't mention it, but I image the date will be a secondary key that will allow sorted for key-range queries

![[Pasted image 20240214083219.png]]

We only allow booking at around 110% capacity, so we can add an if statement into the application for this. 
* SQL supports if statements? Wow

Bonus Question -> What to do if reservation data is too much for DB?
* My guess is sharding or partioning based on hotels, rooms, or even dates
* Can have hot hotels, probably not hot rooms?
* Xu Also mentions that we can add cold storage to reservation history as it's not likely to be accessed often
* He also mentions sharding based on hotels ofc, just hash the IDs

## Concurrency issues

Need to prevent double bookings
* Mutex is my first though. Or a lock
	* We also still want at least a 1 second on the confirmation, I don't think this affects that at all so it's good

Don't let the same user click the book button multiple times
* Gray it out on the client side
	* If user disables JS they can get around this
* Idempotent APIs -> Add idempotency key in the reservation API request
	* Check if the primary key already exists for the reservation


Don't let multiple suers have the same reservation
* Pessimistic Locking
* Optimistic Locking
* Database constraints

![[Pasted image 20240214084207.png]]

Previous SQL query

### Option 1: Pessimistic Locking

Locks at the start of the update so other users can't
* Locks all rows returned by the SQL query

Pros
* Prevents apps from updating data
* Easy to implement
Cons
* Deadlocking can occur when mutliple resources are locked
	* Writing deadlock free code can be hard
	* It's the infinite wait time problem
* Not scalable, lots of time waiting
	* Can use DB server resources

Xu does not recommend this option

#### Option 2: Optimistic Locking

Uses timestamps or version numbers to allow both to update the resources
* Version number is considered better due to server clocks not being in sync

If V2 already exists, don't commit the update and raise an error to User

Pros
* Prevents apps from editing stale data
* Faster than Pessimistic because no dead waiting
* Great when the concurrency is low

Cons
* Performance is bad when concurrency is high and multiple people want to update

## Database Constraints

Immediately increment the inventory, which will deny the option for any other people?

Pros
* Easy to implement
* Works well with low concurrency
Cons
* Similar to optimistic locking
* Users may see available rooms, but it fails when they try to do it
* No version control
* Not all DBs allow constraints, so we may have trouble if we want to migrate in the future


#### Scalability

Because we use SQL, we can't scale that much

1,000 QPS would be tough, but we can still use sharding
* What if we were like booking.com or Expedia and we have way more queries

### Database Sharding

Shard by hotel_Id

I think there's more questions to be raised about sharding by location or even talking about secondary keys, but okay

### Caching

Only the future and current hotel inventory matters

We can use a TTL cache to store inventory, and it'll auto expire

We can add a cache layer on top of the DB too
* not sure what this fully does
* I can only think for popular rooms, but this is only for inventory in his book

Query the cache for all inventory requests instead of the DB

Challenges
* Good for scalability
* Data consistency is now more difficult as we don't have ACID from SQL

Apparently this doesn't matter because the Database will record everything?


### Data Consistency among services

Xu says his approach uses a hybrid monolith / microservice approach.
* Reservation service handles inventory and reservation APIs

Some interviewers will only prefer microservice architecture
* Interviewer may say that each service should only have their own database
* This has a massive struggle with data consistency

![[Pasted image 20240214092530.png]]


Two Phase Commit Strategy
* Slower but will lock nodes

Saga
* Local transactions, if a step fails it undoes everything that was done





