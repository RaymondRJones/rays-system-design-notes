### DNS

### Load Balancer

### Reverse Proxy
Can do blacklisting for IP addresses, caching, and other things along with load balancing

### Application layer
Zookeeper, ETCD, etc.

Add more overhead but can do even more

# Databases

Relational DB's
* Great for ACID
* Can't really scale well 
* Can have writes and read replicas I think

Replicatoin
* Master-slave
	* leader follower where one write and others copy
* master - master
	* CON is that it's loosely consistent
	* conflict resolution is more important
* Improves availability
* Con -> Can have data loss
	* Need to make sure your steps are correct for handling

### Federation
Split databases by functions like
* users DB, forums DB, products DB

### Sharding
Split Database into parts like from A, B, C, D...
It's faster to search for info in A now

Less reads and writes, more cache hits, lower index size

If one shard is down others can be used

Hot partition issue

### Denormalization
You can improve query speed by duplicating data

### SQL Tuning
Not sure what this is
Apparently it's just optimizing SQL queries and knowing it's limits

Avoid expensive joins
Use indexes
Parition tables
Know variable constraints



## NO SQL

More versitile but loses ACID

Cassandra -> Great for writes
DynamoDB -> Great for reads

A ton of specific DBs too
Like timeseries DBs or graph DBs for road maps (Quad Tree)
Even email DBs or search DBs (Elastisearch)

Scale super well

Graph DBs are a thing too, but they aren't used. NoSQL is still better on performance while graph are good for simple queries

Can store massive data unlike SQL

# Cache

A (always?) in-memory server that can be used to initally search things before going to DB
Can put popular requests here so they show up faster

Pros
* Speed
Cons
* Another layer can reduce consistency
* Server has something else to do

# Async

Great for availability, probably bad for consistency

Client doesn't need to wait

Can achieve using a Message Queue like Kafka

### Message Queue
* Built with a Write ahead log as data store and producers / consumers
* Can push messages or pull messages
* Can assign topics
* Also handles retries and exact delivery if needed
* Redis, RabbitMQ, and Amazon SQS are examples
### Task Queue
Celery
Grab tasks, run them on server, delivers results

### Back Pressure

If queue is overloaded, some tasks can risk never being completed

Can reduce queue size and give a 503 error to the client taht tries to add to full queue

#### Communication

7 layers

TCP, UDP

Physical layer with a cable

HTTP layer

gRPC -> Not sure





---
`

```
{"action":"createRoom", "username":"test1"}
```


```
{ "action": "createRoom", "username": "test1", "connectionId": "" }
```


```
{
    "headers": {
        "Host": "rktndgn0fd.execute-api.us-east-1.amazonaws.com",
        "Sec-WebSocket-Extensions": "permessage-deflate; client_max_window_bits",
        "Sec-WebSocket-Key": "9JMGd/gJladA5peel7MQAw==",
        "Sec-WebSocket-Version": "13",
        "X-Amzn-Trace-Id": "Root=1-6609e480-02b7451027099e780e7aa81e",
        "X-Forwarded-For": "181.58.138.178",
        "X-Forwarded-Port": "443",
        "X-Forwarded-Proto": "https"
    },
    "multiValueHeaders": {
        "Host": [
            "rktndgn0fd.execute-api.us-east-1.amazonaws.com"
        ],
        "Sec-WebSocket-Extensions": [
            "permessage-deflate; client_max_window_bits"
        ],
        "Sec-WebSocket-Key": [
            "9JMGd/gJladA5peel7MQAw=="
        ],
        "Sec-WebSocket-Version": [
            "13"
        ],
        "X-Amzn-Trace-Id": [
            "Root=1-6609e480-02b7451027099e780e7aa81e"
        ],
        "X-Forwarded-For": [
            "181.58.138.178"
        ],
        "X-Forwarded-Port": [
            "443"
        ],
        "X-Forwarded-Proto": [
            "https"
        ]
    },
    "requestContext": {
        "routeKey": "$disconnect",
        "eventType": "DISCONNECT",
        "extendedRequestId": "VhCkKF06oAMFc4g=",
        "requestTime": "31/Mar/2024:22:32:32 +0000",
        "messageDirection": "IN",
        "stage": "production",
        "connectedAt": 1711924352723,
        "requestTimeEpoch": 1711924352725,
        "identity": {
            "sourceIp": "181.58.138.178"
        },
        "requestId": "VhCkKF06oAMFc4g=",
        "domainName": "rktndgn0fd.execute-api.us-east-1.amazonaws.com",
        "connectionId": "VhCkKcBNoAMCLAg=",
        "apiId": "rktndgn0fd"
    },
    "isBase64Encoded": false
}

```


```
#set($inputRoot = $input.path('$'))
{
    "action": "$context.routeKey",
    "username": "$inputRoot.username",
    "connectionId": "$context.requestContext.connectionId"
}
```






```
wscat -c "wss://mply1od6ve.execute-api.us-east-1.amazonaws.com/production/" --header "x-api-key: OuhtOfHQSFavV1b1dacWL5rwSbE2d77s6DO57kMc"

```

