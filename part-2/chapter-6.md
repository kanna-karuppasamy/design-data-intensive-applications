# Chapter 6: Partitioning

## Partitioning and Replication

Partitioning is rarely used in isolation; it is almost always combined with replication. Each partition is replicated across multiple nodes for fault tolerance. In a leader-follower model, a single node often serves as the **leader** for some partitions and a **follower** for others to maximize resource utilization.

---

## Partitioning Key-Value Data

The core challenge of partitioning is avoiding **skew** and **hot spots**—situations where one partition receives significantly more load than others, bottlenecking the entire system.

### 1. Partitioning by Key Range

Data is assigned to partitions based on continuous ranges of keys (e.g., A-C, D-F).

* **Pros:** Supports efficient **range scans**. Because keys are sorted within a partition, fetching related records is fast.
* **Cons:** Highly susceptible to hot spots. If keys are based on timestamps, all current writes will hit the same partition (the one for "today").
* **Mitigation:** Use a compound key. For example, prefixing a timestamp with a sensor ID to distribute the write load.

### 2. Partitioning by Hash of Key

A hash function is applied to each key, and the resulting hash determines the partition.

* **Pros:** Distributes data uniformly, effectively "shuffling" skewed data to eliminate hot spots.
* **Cons:** Destroys the ability to do efficient range queries. Nearby keys in the original sort order are scattered across different partitions.
* **Consistent Hashing:** A specific approach to hash partitioning used to minimize data movement during rebalancing (though "hash partitioning" is a more accurate general term for database contexts).

### 3. Compromise: Compound Primary Keys

Systems like **Cassandra** use the first part of a compound key (the partition key) for hashing and the remaining parts (clustering columns) for sorting. This allows for efficient range scans *within* a specific hash bucket.

---

## Partitioning and Secondary Indexes

Secondary indexes are complex because they do not map 1:1 to the primary key partitions. There are two main architectural patterns:

| Feature | Document-based (Local Index) | Term-based (Global Index) |
| --- | --- | --- |
| **Logic** | Each partition maintains an index of only its own data. | The index is partitioned separately from the data, often by value (term). |
| **Writes** | **Fast.** Only one partition needs to be updated. | **Slow.** A single write may update multiple index partitions. |
| **Reads** | **Expensive.** Requires **Scatter/Gather** (querying all partitions). | **Efficient.** Query can be routed to the specific partition containing the term. |
| **Consistency** | High. | Usually asynchronous (eventual consistency). |

---

## Rebalancing Partitions

As load increases or nodes fail, the system must move data between nodes. This is called **rebalancing**.

### Strategies for Rebalancing

1. **Fixed Number of Partitions:** The database is split into many more partitions than nodes (e.g., 1,000 partitions for 10 nodes). When a new node is added, it "claims" partitions from existing nodes. (Used by Riak, Elasticsearch).
2. **Dynamic Partitioning:** When a partition exceeds a size limit (e.g., 10GB), it splits into two. If it shrinks, it merges with a neighbor. This is the B-tree approach. (Used by HBase, MongoDB).
3. **Proportional to Nodes:** The number of partitions is a fixed multiple of the number of nodes. As the cluster grows, the partitions become smaller. (Used by Cassandra).

### Manual vs. Automatic Rebalancing

* **Automatic:** Convenient but dangerous. An automated system might misinterpret a temporary network spike as a node failure and trigger a massive, resource-heavy data transfer, further slowing the system (a "cascading failure").
* **Manual:** Slower but safer. An administrator typically initiates the move, ensuring the system remains stable during the transition.
