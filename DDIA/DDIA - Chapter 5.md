# DDIA Notes Chapter 5

## Replication

Replication is useful when:

- One system goes down, others still work (no Single Point of Failure - SPF).
- Data doesn't change over time, making it easy to replicate.

Distributed databases must handle the challenge of data changing over time using strategies like single-leader, multi-leader, and leaderless, each with its pros and cons.

Replication also has its pros and cons, including considerations like Synchronous vs. Asynchronous Replication, handling failed replicas, and the challenge of developers not understanding *eventual consistency* and checking reads/writes.

### Leaders and Followers

- Each node that stores a copy is called a replica.
- Every write to a database is first sent to the leader, who then forwards it to the followers.
- This setup is also known as active/passive or master/slave.

### Synchronous vs. Asynchronous

- In synchronous replication, a user writes to the database, the leader is notified, updates, acknowledges the user, and tells followers to update.
- Pros: Followers are guaranteed to have the same data eventually.
- Cons: Writes won't work if a follower crashes, and the leader must block all further writes until the follower replica can work.
- Usually, one node is synchronous, and others are asynchronous to avoid system co-dependence. Alternatively, all nodes can be asynchronous, but this may lead to data loss and lowered durability.

### Adding New Followers

- It's challenging to add new followers mid-way because clients are constantly changing the database.
- Locking the database would shut down the application, causing a loss of high availability.
- Instead, you can make a snapshot of the leader and then make all the changes to data that the leader does in the timeframe.

### Node Outages

- The goal is to keep the impact of a node outage as small as possible so the system can keep running.
- For follower failure, followers keep a log of the leader's changes, and when the node restarts, it reads the log.
- For leader failure, you need to detect that the leader has failed, assign a new leader, and redirect followers/info to the new leader. However, this can introduce issues like data loss with asynchronous leaders and the possibility of creating two new leaders (split brain).
- Deciding what timeout is needed for a leader to be declared dead can be challenging, and it might be better to handle these situations manually.

### Implementing Replication Logs

- Replication logs can be implemented by sending SQL Insert/Update/Delete directly to the followers.
- There are challenges with certain SQL statements (e.g., NOW() or Rand()), auto-incrementing columns, and dependencies on current DB data.
- Write-ahead Log (WAL) Shipping, Logical Logs, and Trigger-Based Replication are alternative methods with their own advantages and disadvantages.

### Problem with Replication Lag

- In read-heavy applications, followers can provide faster reads, but what happens if a follower isn't up to date when a read happens?
- Eventual consistency can usually be achieved by waiting.

### Reading own Writes

- Applications that allow users to see what they've written right after writing have various strategies, including reading from the leader for small edits and tracking replication lag for many edits.
- Dealing with multiple devices and data centers adds complexity.

### Monotonic Reads

- Monotonic reads are essential to ensure consistency when users make read requests to different followers.

### Consistent Prefix Reads

- Consistent prefix reads are important to guarantee that writes occur in the right order when being read.

### Multi-Leader Replication

- Multi-leader replication is a good alternative when connection loss to a leader is a concern, although it adds complexity.
- It's useful for multiple data centers, reducing latency, and providing redundancy for data center outages.
- However, it introduces issues like data racing and write conflicts between data centers, especially with triggers, auto-increment keys, and integrity constraints.

### Handling Write Conflicts - Data Racing

- Handling write conflicts (data racing) depends on the replication setup, involving locks and waiting in a single setup or asynchronous writes with conflict resolution in a multi-set up.

### Multi-Leader Replication Topologies

- Deciding how leaders should send information to other leaders is a key consideration.

### Leaderless Replication

- Some database systems, like Dynamo, Cassandra, Riak, and Voldemort, use leaderless replication.
- Handling a follower being down involves completing writes on other followers and reading from all followers to compare version numbers.
- Failed nodes catch up by updating their version during reads and through background scans.

### Quorums and Consistency

- [Details on quorums and consistency are not provided in this summary.]

