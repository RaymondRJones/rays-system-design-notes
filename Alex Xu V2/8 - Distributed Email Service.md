# TLDR
* Questions
	* How does caching structure work
		* 



# Problem
Large-scale email services like Gmail, Outlook, and Yahoo

Gmail has about 1.8B active users

# Step 1 - Understanding the Problem and Establish Scope

Very important because email services are a complex system with many features. Need to narrow down what the interviewer cares about

How many people use the product?
* 1B

I think these features are important. Is this good?
* Authentication
* Send / Receive Emails
* Fetch all emails
* Filter emails by read an unread
* Search emails by subject, sender, and body
* Anti-spam and anti-virus

Answer -> Good list don't worry about Authentication

How do users connect with mail servers
* Native clients that use SMTP, POP, IMAP, and specific protocols

Can emails have attachments?
* Yes

#### Non Functional Requirements

Availability
Eventual Consistency
Reliability
* Not Duplicate emails
* Don't lose emails
Flexibility and extensibility
* Be able to add new features and enhance components
* POP and IMAP have very limited functionality

#### Back of Envelope Estimations

1B users
* Sending emails
	* 1B/day -> 10^9 / 10^5 seconds -> 10,000 QPS
	* 10 emails per day
	* 10B -> 100,000 QPS
* Reading emails
	* 3B/day -> 30,000 QPS
	* Average receives is 40 emails per day
	* 400,000 QPS
* Average email size is like 50kb with the meta data
	* 1B users x 40 emails/day x 365 x 50KB -> 1600 x 1B x 50
	* 16,000 x 10 ^9 --> 16 10^12B ---> 160 PB every year
		* 20% have a 500KB attachment size so we have to add that
	* I might have messed that earlier math up, but Xu gives
		* 1,460PB of data



# Step 2 -> Propose High Level Design and Get Buy-in

### Email 101 - What is it

SMTP -> Standard protocol for sending emails from one mail server to another

POP -> Standard protocol for receiving and downloading emails from a remote mail server to a local email client. Once downloaded, they are deleted from the server.
* Requires clients to download the entire email

IMAP -> Standard protocol for receiving emails fro a local email client
* Downloads the message when you click it. Emails aren't deleted so
* Can access from different devices

HTTPS -> Not a mail protocol, but can be used to access your mailbox
* Specifically for web-based email, like Microsoft Outlook

### DNS

Looks up the mail exchange record (MX record) for a domain.

DNS will have several mail server options typically for a given domain
* Tries each one if they fail

#### Attachments
* Base64 encoding. Usually a size limit for attachments
* Gmail and Outlook have 25MB and 20MB as of June 2021
* Number is highly configurable and varies from individual and corporate accounts.
* Multi-purpose Internet Mail Extension (MIME) is a specification that allows the attachment to be sent over the internet

### Traditional mail servers

![[Pasted image 20240214164628.png]]


For storage, maildir was typically used. It stored it as a file system of users

![[Pasted image 20240214164800.png]]

This file structure became a bottleneck as users continued to grow. What is a server dies? What about back ups?

Need a better distribution layer

### Distributed Mail Servers

Design to support the modern use cases and solve problems of scale and resilency

#### Email APIs

SMTP / POP / IMAP APIs for native mobile clients
SMTP communications between sender and receiver servers
RESTful API over HTP for full featuresd and interactive web-based emal applications

Xu will only cover some of most important APIs for webmail. HTTP

1 -> POST /v1/messages
* Send to recipients in To / Cc / Bcc
2 -> GET v1/folders
* Get folders for an account
3 -> GET /v1/folders/{:folder_id}/messages
* Return all messages under a folder. It's simple but ideally would support pagination
4 -> GET /v1/message/{:id}
* get all info for a specific message

### Distributed Message Architecture

How do we store messages? How to we sync data across servers?  How to keep email from flagged as spam?

![[Pasted image 20240214170103.png]]

Webmail -> web browsers to send and receive emails

Web servers -> login, sign up, profiles, sending email, loading folders, etc.

Real time servers -> Provide real-time updates to the user
* Stateful but support real-time connection

Metadata database
* subject, body, from user, to user, etc.
* video_link_to_S3_Buckets

Attachment store
* AWS S3
* Mentions that Cassandra has a practical limit of 1MB
	* Can't use a row cache as attachments take too much memory

Distributed Cache
* Recent emails that have to load often
* Redis is good

Search store
* Uses Inverted Index to search through text

### Email Sending flow

![[Pasted image 20240214170653.png]]

We use basic email validation on the web servers and put in error queue

Outgoing Message Queue

SMTP Outgoing Serbvice
*  checks message spam / viruses
* Moves to SENT folder for user
* Sends to the user via internet

### Email Receiving Flow

![[Pasted image 20240214171053.png]]

1 -> Incoming emails at SMTP Load balancer
2 -> Distributes traffic among SMTP servers
* Email acceptance policy
* Invalid emails can be bounced to avoid unnecessary processing
3 -> If attachment on email is too large, we can put in S3 bucket
4 -> Emails are put into the incoming email queue
* Serve as buffer, pub/sub can help reduce volume
5 -> Mail Processing workers
* Filtering spam and viruses, etc.
* Valid
6 -> Email stored into databases

7 -> If connection / receiver online, email pushed to real-time servers

9 -> offline users will grab when they come online

10 -> Web servers pull new emails from storage and send to client

# Step 3 - Design Deep Dive

#### Metadata database
* Email header is small
* Body is small to big, but only read once typically
* Most of the mail operation are for individual users
* Data recency impacts data usage. Users only read recent emails
* Data loss is not acceptable

Database options
* Relational
	* Can search through emails efficiently
	* has ACID
	* Emails are typically a few KB
	* Not a good fit, SQL doesn't work for large data
* Distributed Object Storage
	* AWS S3
	* Hard to support marking emails as read
	* And search is hard
* NoSQL
	* Google Big Table is used by Google
		* Not open source
	* Cassandra could work, but nobody has done that
In an interview you won't have time to design your own propiertary database
* Just explain the characteristics that a database should have
* Strong consistency
* Column can be single digit of MB
* Highly available and fault-tolerant
* Easy to create incremental back ups

### Data Model

We can parition databases based on user_id
* Struggles with having users with multiple accounts, but that's okay for this problem scope

Primary key has two components -> Parition key and clusering key
* Cluster key is used to sort data within paritions
* Secondary keys just add another index to a column to make searching faster
	* Not sure how this works though

Need to support the following queries
* Get All folders for user
* Display all emails in a folder
* Create / Delete / Get a specific email
* Fetch all unread emails or read emails
* Bonus -> Get conversation threads
Query 1: Get all folders for user

![[Pasted image 20240215071650.png]]

Query 2: Display all emails for a specific folder

We can sort the emails by timestamp. Partition them by their user_id and folder_id
The email_id can be the TIMEUUID to use for sorting.

This let's us find the unread emails fastest

emails
* folder_id
* user_id
* timestamp
* preview
* is_read
* subject


Query 3: Create / Delete / Get an email

![[Pasted image 20240215072244.png]]

Query 4: Fetch all read or unread emails

![[Pasted image 20240215072350.png]]

Xu's solution said no SQL databases earlier though

NoSQL only supports queries on partition and cluster keys
* is_read is not one of these keys, so it will reject it

One option is to fetch the entire folder and filter it ourselves but that's inefficient

NoSQL can use Denormalization
* Create two tables instead of having the attributes on a column
* Read Emails
* Unread Emails


### Consistency Trade Off

Distributed Databases that use replication for high availability must make a tradeoff between consistency and availability. Correctness is very important for email systems, so we want a single primary for any given mailbox. The mailbox won't be accessible to clients during a crash / failover, but we want consistency

## Email Deliverability

It's easy to send emails. hard to get them not flagged as spam

Use dedicated IPs from trusted sources
Classify emails -> Send marketing emails from one IP address
Email sender reputation -> Slowly build a good reputation for Google
* This is probably how social media platforms work
Ban spammers quickly
* Ban accounts of people who spam
Feedback processing
* Keep complaints low
* Hard bounce - > Invalid email addresses
* Soft bounce -> Busy servers
* Complaint buttons
Authenticate emails

## Search

Search depends on context. Emails have a lot of different attributes that can be searched for.

We're constantly reindexing when a user sends, receives or deletes emails (What does this mean??)

This means the search for emails has a lot of writes

![[Pasted image 20240215073936.png]]


### Option 1 -> Elastic Search

![[Pasted image 20240215074230.png]]

Reindexing can be done on the database, and we can query using a synchronous call to find the mail the user wants

Challenge of Elastic search is keeping primary email store in sync with it


#### Custom Search Solution

Designing an email search engine is very complicated and out of scope

Xu will talk about the disk I/O Bottleneck that one could face trying to do this

Users could easily have half a million emails. Attachments are at the PB daily level. So many writes.

Disk I/O is the main bottleneck

LSM trees are good for writes

![[Pasted image 20240215074618.png]]

## Scalibility and Availability

Scales horizontally and can be made available using replication

#### Follow ups

Email Security and Compliance

Fault Tolerance


