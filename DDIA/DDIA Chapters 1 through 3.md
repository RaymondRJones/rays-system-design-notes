
Data Intensive Application
	Data as primary challenge
	Quantity of Data / Complexity of Data / Speed of Data

Lots of Buzzowords like AWS / NOSQL / etc.
	Choosing from these means understanding tradeoffs
Book details how to navigate and select from all these options

------------
PART 1
------------
Reliabilitiy
Scalability
Maintability


Databases / Caches / Search Index / Stream processing / Batch Processsing
	SP is filtering data by key words
	Batch Processing is crunching large amounts of data

These can all be considered Data Systems and they can interact with each other to help a overall application
	
Reliability
	User errors or Mistakes are handled
	Performs as user expects
	Performance meets required case
	Prevents unauthorized access
	Functional / Non-Functional Requirements*

Hardware Errors
		Problems with hardware can arise, in sources that want to be highly avaiable
			RAID / Redudancy is added. But this increases rates of hardware failure
			Common for AWS to be down
	Software Errors
		Cause bigger system errors than hardware errors
		No real solution, although testing, process isolation, and crash/restart can help
			Measuring / Monitoring / Analyizing system behavior in production
		Test at all levels -> Unit / Integration / Manual / Regression
		Roll back capabilities
	Generally very important to reduce loss of money (Or catatrophic nuclear failure)
		Can cut corners but need to be conscious of risks
SCALABILITY
	What will the service look like in the future? What happens if data expands? How is it handled?

Discover what current load is, then look for ways to optimize*
Twitter example with updating timeline tweets at WRITE vs. VIEW

Once load is found, examine what happens when it increases (Measure Throughput or time)

latency vs. response time (time until request finishes vs. time until user sees change)
		Median is better than average for estimating time data
		Percentiles can also tell a story -> higher percentile (tail latency) usually serve bigger data
	Service Level Objectives (SLOs) and Service Level Aggrements (Contracts the define expected performance)
	Queueing delays and waiting for 1 slow application request can stall everything else

How to improve architecture scalability
		Usually have to redesign it as it scales (Load 1 design does not work for Load 10 design)
		Vertical scaling vs. Horizontal Scaling (shared-nothing / scaling out)
		Elastic systems for unpredictable traffic
	There is no magic key, need to evaluate types of distributed systems
MAINTAINABILITY
	People dislike working with legacy software, but it's important to reduce pain
	Operability / Simplicity / Evolvability 
	
Operability involves making life easier for operators (People who monitor / keep software up to date)
		Data systems that have good visibility for this is preferred
		Good Documentation / default behavior / Avoid dependency on 1 machine(!!!) 
	Simplicity 
		OOP / consistent naming / Avoiding tangled dependencies


--------------
CHAPTER 2 - DATA MODELS AND QUERY LANGUAGES
--------------
Data models are important part of applications
Very difficult to master the intricacies of one data model (There's 20+ books on relational modeling)
	This chapter will go over Relational / Document / some graph based models

RELATIONAL VS. DOCUMENT MODEL
	SQL as most popular relational DB model
	Developed in 1970s and was overall best among competitors, also has good general use cases
	NOSQL rose to reduce difficulties from SQL
		Scalability, simpler to code than SQL, specialized query operations, Open Source
		Likely that NOSQL co-exist with SQL 
	Object-Relational Mismatch (Tables don't map to objects, code needs to be made for this)
		JSON representations don't have this problem (However, it still has other issues)
	IDs. vs Strings (IDs are useful because they remain consistent / avoid ambiguity, human language changes)
	
	Many to One vs. One to Many
		Using Joins on JSON is hard, maybe can reference objects instead of strings

DOCUMENT MODEL
	In 1970s, IBM's IMS was a hierarchical / tree model that had many similarities with JSON
	Eventually faded away when SQL grew
	Network MODEL / CODASYL -> Struggled if the path for data wasn't known
	Relational DB was another solution -> Allows data without lookups / paths... Query optimizations
	
Code Complexity
	Document good at One to Many (Lack of joins means Many to Many is difficult)

Schema Flexibility
	No schema enforced means any document can contain anything
	Documents have a schema-on-read, relational has schema-on-write that's enforced
	What if an application wants to change the format of data? Document needs to write new documents with new fields
		Relational has to use ALTER query

	Documents are great if data stored can have different fields, Schema-enforced relational DB would be hard here

Data Locality
	All the data is loaded at once (Faster if all data is being used)
Convergence of Document / Relational
	XML is allowing querying / indexing.... PostgreSQL is allowing JSON support.. Others soon to follow?

QUERY LANGUAGES
	Imperative -> most common 
	Declarative programming offers super fast code by telling computer what type of data to get
	Map Reduce Querying -> hybrid (MONGODB)

GRAPH-LIKE MODELS
	Document good for ONE-to-MANY or No-relationships... Relational good for many-to-many
	When the many-to-many is too big -> Use graphs
	Social Network // Web graph // Rail / road network

Property Graph Model (Neo4j, Titan, InfiniteGraph)
	Vertex has ID / outgoing edges / incoming edges / properties (key-val pairs)
	Edge has ID / head(ends) / tail vertex (where edge starts) / label for relationship / properties

No restrictions for vertex connections // Can easily find incoming + outgoing edges for a vertex // Labels for relationships can store info on graph
	Has good evolvability due to flexible nature
	
Cypher query language for graphs

Triple-Store Model

SUMMMARY
	Brief overview of the big field data models
		Unmentioned Models -> Genome data // Particle physics // Full - Text Search
------
Chapter 3 - Storing and Retrieving Data (Reads and Writes)
-------

Simple method is to append to end of file. And search entire file when reading (slow)

Hash Index
	Mapping data at offsets so you can skip more of the file
	k1 -> offset 1, key2 -> offset 64 (better than searching entire file)

	These files can be segmented to allow new files to be created when space runs out
		Compaction (Throwing away duplicate keys in log)
	Details of implmentation
		Store using byte file
		Append special char to bytes that need to be deleted
		Crash recovery
			Store snapshots that can be reloaded
			Partially written records deleted
			Writer locks file
	Benefits -> Appending is fast, crash recovery is simple
		removing old segments removes need of fragmentation
	Limitations -> hash-table must fit in memory
		Range queries are difficult / slowish

SSTables and LSM-Trees
	Sorted String Table (SSTables)
		Instead of appending to end, keep in sorted order (Allows faster searching)
	How to maintain sorted order?
		B - Trees work. In-Memory works too using a tree structure like Red-Black / AVL
		1. Insert in sorted order
		2. Check if memtree is bigger than some capacity...Make new one if it is
		3. on Read, search memtable, then check disk-segment, and older segments
		4. Occasionally run merge / compaction in background to delete files/overwrite values
	Handling Crashing
		Memtable is lost if it crashes...Prevent loss by append every memtable write to a stack
	
	LSM - Trees
		Process above is used in LevelDB and RocksDB
		Indexing structure called Log-Structured Merge-Tree
		Seem to be exact same as SSTables
		Optimizing -> Misses are expensive
			Fix this by using Bloom Filter(Approximates set contents)
		Strategies for compacting/merging
			Size-Tiered / Leveled Compaction. HBase(level)/Cassandra(does both)
			Newer / smaller SStables are merged into older/larger tables (Size-tiered)
			Key range is split into smaller SSTables and older data moved into levels
		LSM keep a cascade of SSTables that are merged in background. 
		Simple and Effective
B-Tree
	One of most common index
		References point to other References in a range of numbers
		Uses fixed blocks / Pages (4KB usually)
		
		During update -> Search ranges for the key
		During insert -> Search ranges until at the end, and add it (If size too big, split list in half)
			Then update parent
	Can watch a youtube video to learn more

LSM-Tree vs. B-Tree
	LSM has less writes. B-Trees need to write at least twice
		(Once for WAL log in case of crash, Once on tree)
		Some storage engines write same page twice to prevent power outage partial write
	LSM rewrite during compaction / merging phase (Write amplification)
	Write heavy application will notice the difference
	LSM is compressed better, yielding smaller files on disk

	---
	BAD of LSM
	---
	Compaction can be expensive
	Disk bandwidth expands bigger database gets
	Compaction could possibly not keep up with the write requests
	
	B-Trees provide consistently good performance. Each key exists in one place wheras LSM might have multiple copies
	Transaction isolation can be done by locking ranges

OTHER STRUCTURES
	Primary key -> Uniquly identifies a row in SQL/relational
	Secondary index -> Important for joining data
Heap file -> stores rows for indexing... Avoids duplicating data when multiple secondary indexes present
	Index referes to location
Multiple Columns
	What if you want a key to have multiple values?
	Concatenated index -> rule is set to append X columsn when key/index is found
		Multiple-dimensional like below can't be done in a B-Tree / LSM

		SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
			AND longitude > -0.1162 AND longitude < -0.1004;

Ful-text-search / fuzzy indexes

In Memory Databases
	Don't need to write to a hard disk, everything in memory
	Can use data models that are complicated to use with a hard disk

	Anti-caching approach, a type of virtual memory can be used by evictng LRU cache and putting in disk
-------	
Transaction Processing vs. Analytics
-------
Online Transaction Processing (OLTP)
	Standard CRUD applications, writing data into database and reading it. Blogs / Old Websites / Banking / etc.
Online Analytic Processing(OLAP) -> What was revenue for Jan -> June... / Queries to improve business
	Reading large amounts of data and taking a small part of it
Now, we are moving away from using SQL for both... Data Warehousing for OLTP

Data Warehousing
	Give the analytics people their own read-only database
	Improves security / isolates processing for the big OLTP data
		OLTP needs to be highly available and low latency
	ETL (Extract, Transform, Load) -> Moving OLTP data to OLAP
	Small businesses usually don't know about OLAP, the small data works with pure relational
		Data warehouses can be optimized for analytics patterns
			indexing algorithms don't help OLAP queries
	Data Warehouse is typically a relational database for the queries
	STAR SCHEMA -> WHERE/WHEN/HOW, one central table that branches off into these
		dimension tables**
		Data warehouses have very wide tables and fact tables have over hundreds of columns
		Dimension tables can also be wide
	OLTP databases are usually row-oriented