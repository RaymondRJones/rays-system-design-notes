# DDIA Notes Chapter 6 - Sharding

Chapter 5 discussed replication -> Having multiple copies of data on different nodes

## Sharding / Partitioning

Sharding or partitioning allows breaking up data into partitions, where each partition is essentially a small database. This approach is used to distribute a large dataset across many processors, leading to faster lookups for big data.

Usually combined with replication, although the strategy is the same as in Chapter 5, so replication won't be covered.

---

### Key Range Partition

- Like an Encyclopedia, group the data.
- Inside each partition/page, the keys can be sorted to improve searching speed.
- Downside is that data is not evenly distributed, leading to hot spots.

### Partition by Hash

- Hash function evenly distributes the data.
- Also called consistent hashing, although the vocabulary is awkward in some places.
- Range queries aren't supported, and there is no sorted order to search.

---

### Skewed Workloads and Relieving Hot Spots

Hash doesn't fix everything; sometimes there are hot keys (like a celebrity on Instagram). Keys can be split into parts to ease the load, such as adding two random numbers to the celebrity ID to yield more partitions. This splits the writes but means reads need to check all 100 different fields.

---

### Secondary Indexes

Another index is needed, like occurrence of User 123 or the number of cars with the color red. Relational databases are good at this, but partitioning these indexes is difficult.

---

### Rebalancing Partitions

Evenly distributing the load of all partitions as data changes over time is essential as systems crash, the dataset increases, RAM increases, queries increase, and CPU increases. Hash mod is not ideal because it requires data to be moved around after it changes (e.g., Mod 10 -> Mod 15).

- Fixed # of Partitions: Start with 100 and don't change it. New nodes will steal more partitions from all other nodes until balanced. This is used in Riak, Voldemort, ElasticSearch.

- Dynamic Partitioning: No need to waste space; allow it to adjust itself.

