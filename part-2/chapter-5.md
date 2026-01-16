# Chapter 5: Replication

Replication involves keeping a copy of the same data on multiple machines connected via a network. This chapter explores how to handle changes to replicated data, primarily focusing on the **leader-based** approach and the trade-offs involving consistency, availability, and performance.

## Why Replicate?
Distributed systems replicate data for three primary reasons:
* **High Availability:** Keeping the system running even if one machine (or several) fails.
* **Disconnected Operation:** Allowing an application to continue working when there is a network interruption.
* **Latency:** Placing data geographically closer to users for faster access.
* **Scalability:** Increasing the volume of read queries the system can handle (read-throughout).

---

## Leaders and Followers (Master-Slave)
The most common approach to replication is **leader-based replication**. In this model, nodes are assigned specific roles:

1.  **The Leader (Master/Primary):** Every write to the database must be sent to the leader. The leader writes the new data to local storage and sends the change to all its followers as part of a **replication log** or change stream.
2.  **Followers (Read Replicas/Slaves):** These nodes are read-only from the client's perspective. They apply the leader's log to update their local copy in the exact same order as the leader processed them.



### Synchronous vs. Asynchronous Replication
The timing of when a write is considered "successful" is a critical configuration choice:

* **Synchronous:** The leader waits for the follower to confirm it received the write before reporting success to the user.
    * **Pro:** The follower is guaranteed to have an up-to-date copy.
    * **Con:** If a synchronous follower fails or the network lags, the leader must block all writes, crashing the system's "write" availability.
* **Asynchronous:** The leader sends the message but does not wait for a response.
    * **Pro:** The leader can continue processing writes even if all followers fall behind.
    * **Con:** If the leader fails, any writes that were not yet replicated are lost forever.
* **Semi-synchronous:** A practical hybrid where one follower is synchronous and the others are asynchronous. If the synchronous follower becomes slow, another is promoted to take its place.

---

## Handling Node Outages
The goal is to achieve high availability with minimal manual intervention.

### Follower Failure: Catch-up Recovery
Followers keep a local log of data changes. If a follower crashes, it identifies the last transaction processed before the fault and requests the remaining log from the leader once it recovers.

### Leader Failure: Failover
Promoting a follower to leader is a complex, multi-step process:
1.  **Detection:** Usually done via heartbeats and timeouts (e.g., if a node doesn't respond for 30 seconds, it's assumed dead).
2.  **Election:** Choosing a new leader, typically the replica with the most up-to-date data.
3.  **Reconfiguration:** Directing clients to the new leader and ensuring the old leader (if it returns) becomes a follower.

**Failover Hazards:**
* **Split Brain:** Two nodes both believe they are the leader. Both accept writes, leading to data divergence and corruption.
* **Discarding Writes:** In asynchronous systems, the new leader may lack the old leader's final writes. Discarding these writes breaks durability expectations.
* **Timeout Tuning:** If timeouts are too short, temporary load spikes trigger unnecessary failovers; if too long, the system stays down longer than necessary.

---

## Implementation of Replication Logs
There are four major technical methods used to ship data from the leader to followers:

1.  **Statement-based Replication:** The leader logs every write request (e.g., `INSERT`, `UPDATE`) and sends it to followers.
    * *Risk:* Nondeterministic functions (e.g., `NOW()`, `RAND()`) or autoincrementing columns can cause data to diverge across replicas.
2.  **Write-Ahead Log (WAL) Shipping:** The leader sends the raw disk-block changes used by the storage engine.
    * *Risk:* Closely coupled to the storage engine's version. You cannot easily perform a zero-downtime upgrade if the leader and follower must run the exact same software version.
3.  **Logical (Row-based) Log:** Decouples the replication log from the storage engine internals. It describes writes at the row level (e.g., "In Table X, row Y was changed to Z").
    * *Benefit:* Easier for external applications to parse (useful for **Change Data Capture**) and allows different DB versions on different nodes.
4.  **Trigger-based Replication:** Handled at the application level via database triggers.
    * *Benefit:* Highly flexible (e.g., only replicating a specific subset of data).
    * *Risk:* Significant performance overhead and higher bug potential.

## Problems with Replication Lag

In a **read-scaling architecture**, you can increase capacity for serving read-only requests by simply adding more followers. However, this realistically only works with **asynchronous replication**. If the replication is synchronous, a single node failure makes the whole system unavailable for writes.

The trade-off for this scalability is **Eventual Consistency**. While followers will "eventually" catch up, the **replication lag**—the delay between a write on the leader and its arrival at a follower—can vary from fractions of a second to several minutes under high load or network stress.

### 1. Reading Your Own Writes
A common user experience issue: a user submits data (write to leader) and immediately views it (read from a stale follower). The user sees an old version of their data and assumes it was lost.



**Solution: Read-after-write Consistency**
To guarantee a user always sees their own updates:
* **Leader-based Reads:** Always read things the user *might* have modified from the leader. 
    * *Example:* A user's own profile is read from the leader; other users' profiles are read from followers.
* **Time-based Rules:** If the user can edit almost anything, track the time of the last update. For a window (e.g., 1 minute) after a write, perform all reads from the leader.
* **Logical Timestamps:** The client remembers the timestamp/Log Sequence Number (LSN) of its latest write. The system ensures the replica serving the read has caught up to at least that timestamp.

**Cross-Device Challenges:**
If a user switches from a laptop to a mobile app, they expect to see the same state. This requires:
* **Centralized Metadata:** The "last write timestamp" cannot be stored locally on the device; it must be managed in a central metadata store.
* **Datacenter Routing:** If different devices route to different datacenters, the system must ensure all devices belonging to a user are routed to the same leader-hosting datacenter for consistency.

---

### 2. Monotonic Reads
This anomaly occurs when a user sees things "moving backward in time." This happens if a user makes successive requests to different followers—one fresh, and one lagging.



**Solution: Sticky Routing**
* **Mechanism:** Ensure that each user always makes their reads from the same replica. 
* **Implementation:** Choose the replica based on a hash of the user ID rather than random load balancing. 
* **Downside:** If that specific replica fails, the user must be rerouted, potentially causing a one-time "time jump" backward.

---

### 3. Consistent Prefix Reads
This is a violation of **causality**. If a sequence of writes happens in a specific order, anyone reading those writes must see them appear in that same order. This is a significant problem in **partitioned (sharded)** databases.



* **The Problem:** If the database is partitioned, different partitions operate independently. There is no global ordering of writes. An observer might see an effect (an answer) before they see the cause (the question) if the "answer" partition replicates faster than the "question" partition.
* **Solution:** Ensure that any writes that are causally related are always written to the **same partition**. Alternatively, use algorithms that explicitly track causal dependencies.

---

## Solutions for Replication Lag

While application code *can* work around these issues (e.g., forcing certain reads to the leader), it is complex and error-prone. 

> **The Role of Transactions:** Transactions exist so that application developers don't have to worry about these subtle replication issues. A database that "does the right thing" allows the application to stay simple.

While many distributed systems have abandoned transactions in favor of performance, the trade-offs are more nuanced than simple "consistency vs. availability."

## Multi-Leader

While single-leader replication is common, it has a significant bottleneck: every write must go through a single node. If that node is unreachable, the system cannot accept writes. **Multi-leader replication** (also called master-master or active-active) allows multiple nodes to accept write requests, acting as both a leader to clients and a follower to other leaders.

---

### Use Cases for Multi-Leader Replication

It is rarely recommended within a single datacenter due to complexity, but it is highly effective in specific scenarios:

### 1. Multi-Datacenter Operation

In this setup, each datacenter has its own leader. Within each datacenter, leader-follower replication is used; between datacenters, leaders replicate changes to each other.

* **Performance:** Writes are processed in the local datacenter, hiding inter-datacenter network latency from the user.
* **Tolerance of Outages:** If one datacenter fails, others continue to operate independently.
* **Network Reliability:** Can handle temporary internet interruptions better because it relies on asynchronous replication.

### 2. Clients with Offline Operation

Mobile apps or desktop tools that must work without an internet connection (e.g., a calendar app) are essentially multi-leader systems.

* Every device acts as a local "leader" (accepting writes).
* Syncing is the process of asynchronous multi-leader replication between the device and the server.
* Replication lag can be hours or even days.

### 3. Collaborative Editing

Applications like Google Docs or Etherpad allow multiple users to edit a document simultaneously.

* To avoid waiting for a lock, the unit of change is often as small as a single keystroke.
* Each user's local client acts like a leader, applying changes immediately and replicating them asynchronously to others.

---

## Handling Write Conflicts

The primary challenge of multi-leader replication is that **write conflicts** are inevitable. If two users change the same record in different leaders simultaneously, the data will diverge.

### Conflict Detection: Synchronous vs. Asynchronous

* **Single-Leader:** Conflicts are prevented by blocking or aborting the second writer (synchronous).
* **Multi-Leader:** Both writes succeed on their respective leaders. The conflict is only detected later during replication (**asynchronous**).
> **Note:** Making conflict detection synchronous defeats the purpose of multi-leader replication (independence of replicas).



### Conflict Avoidance

The simplest solution is to ensure that all writes for a specific record always go to the same leader.

* *Example:* Routing all requests for a specific user to a "home" datacenter based on their ID.
* *Risk:* If the datacenter fails or the user moves, you must change the designated leader, which re-introduces the risk of concurrent writes.

---

## Converging Toward a Consistent State

To ensure all replicas eventually contain the same data, the system must resolve conflicts in a **convergent** way.

| Strategy | Description | Risk |
| --- | --- | --- |
| **Last Write Wins (LWW)** | Each write has a timestamp; the latest one is kept. | Prone to data loss due to clock skew or lost intermediate writes. |
| **Replica ID Precedence** | Writes from a higher-numbered replica always "win." | Also implies data loss. |
| **Merge Values** | Concatenate or combine values (e.g., "Title B/C"). | Resulting data may be messy or nonsensical. |
| **Custom Logic** | Use application code to resolve the conflict. | Increases complexity and development overhead. |

### Custom Conflict Resolution Logic

Most tools allow developers to write resolution logic:

1. **On Write:** Triggered as soon as the DB detects a conflict in the replication log. It runs in the background and cannot prompt the user.
2. **On Read:** Conflicting versions are stored (multi-versioning). When a user reads the record, the app detects multiple values and asks the user (or uses logic) to pick one.

---

## Automatic Conflict Resolution

Because custom code is error-prone, modern research focuses on algorithmic resolution:

* **Conflict-free Replicated Datatypes (CRDTs):** Data structures (sets, maps, counters) that automatically resolve conflicts in a mathematically sound way.
* **Mergeable Persistent Data Structures:** Similar to Git; they track history and use three-way merges.
* **Operational Transformation (OT):** The algorithm behind Google Docs, specifically designed for concurrent editing of ordered lists (like text).

> **What defines a conflict?** While overlapping field updates are obvious, some are subtle—like two people booking the same room at the same time on different leaders. These "semantic" conflicts often require cross-partition coordination.

This section finishes the discussion on **Multi-Leader Replication** by focusing on how the leaders are physically and logically connected to propagate writes.

---

## Multi-Leader Replication Topologies

A **replication topology** describes the communication paths writes take to move from one node to another. In a two-leader setup, the path is direct. However, as the number of leaders increases, the complexity of the network increases as well.

### Common Topology Types

There are three primary ways to organize these connections:

| Topology | Structure | Characteristics |
| --- | --- | --- |
| **Circular** | Each node receives writes from one neighbor and forwards them to another. | Used by default in some systems like MySQL. Vulnerable to single-node failure. |
| **Star** | One central root node forwards all writes to the other peripheral nodes. | Easy to conceptualize but creates a central bottleneck and single point of failure. |
| **All-to-All** | Every leader sends its writes to every other leader. | Most resilient to node failures but suffers from message ordering issues. |

### Key Technical Challenges

#### 1. Preventing Replication Loops

In circular and star topologies, a write may pass through several intermediate nodes. To prevent a write from circulating forever, systems use **unique identifiers**:

* Each node has a unique ID.
* Every write in the replication log is tagged with the IDs of the nodes it has already passed through.
* If a node receives a write tagged with its own ID, it ignores it to break the loop.

#### 2. Fault Tolerance

The circular and star topologies are fragile. If a single node fails, it can break the "chain" of communication for the entire cluster. Reconfiguring these topologies often requires manual intervention. In contrast, **all-to-all** topologies are more robust because data can travel along multiple paths, but they introduce a new problem: **causality**.

#### 3. Message Ordering and Causality

In an all-to-all network, some network links might be faster than others. This can cause a later write (like an **update**) to arrive at a replica before the initial write (the **insert**) it was based on.

* **The Issue:** Simply using system timestamps is insufficient because clocks across different machines are never perfectly synchronized.
* **The Solution:** To maintain the "happens-before" relationship, systems must use more advanced techniques like **version vectors**, which track the history of changes across the distributed system.

> **Warning:** Many existing multi-leader tools (like older versions of Tungsten Replicator or PostgreSQL BDR) do not provide robust causal ordering or conflict detection by default. It is critical for developers to verify the specific consistency guarantees of their chosen database.

---
This final section of Chapter 5 explores **Leaderless Replication**, often referred to as **Dynamo-style** (after Amazon's Dynamo system). Unlike leader-based models, this architecture allows any replica to accept writes directly, providing unique trade-offs for availability and consistency.

---

## Leaderless

In a leaderless system, the concept of a "primary" node is abandoned. Clients send their writes to several replicas in parallel, and a coordinator (or the client itself) manages the responses without enforcing a global ordering of writes.

### Writing and Reading When a Node Is Down

In leaderless replication, there is no failover. If a node is offline, it simply misses the writes.

* **Quorum Writes:** A write is considered successful if it is acknowledged by a minimum number of replicas ().
* **Quorum Reads:** To ensure the latest data is retrieved, the client sends read requests to several nodes () in parallel. Since different nodes might return different versions, the client uses **version numbers** to determine which value is the most recent.

#### Keeping Replicas Up-to-Date

Two main mechanisms help a returned node catch up with missed writes:

1. **Read Repair:** When a client reads from multiple nodes and detects a stale value, it writes the newer version back to the stale replica. This is effective for frequently read data.
2. **Anti-Entropy Process:** A background process that constantly compares data between replicas and copies missing data. Unlike replication logs, this does not preserve write order and may have significant lag.

---

### Quorums for Consistency

The reliability of leaderless systems depends on the parameters  (number of replicas),  (minimum write votes), and  (minimum read votes).

> **The Quorum Condition:** As long as , we expect a read to return the latest value because the set of nodes written to and the set of nodes read from must overlap.

* **Common Setup:** Usually  is an odd number (3 or 5), with  (rounded up).
* **Limitations:** Even with , "strict" consistency isn't guaranteed. Edge cases include:
* Concurrent writes (ordering is ambiguous).
* Failed writes that succeeded on some replicas but not enough to reach .
* Restoring a node from a stale backup.

### Sloppy Quorums and Hinted Handoff

In large clusters, a network partition might prevent a client from reaching the  "home" nodes for a specific key.

* **Sloppy Quorum:** The system accepts writes and stores them on any available nodes that aren't the designated home nodes.
* **Hinted Handoff:** Once the network issue is resolved, the temporary nodes hand the data back to the original home nodes.
* **Impact:** This significantly increases **write availability**, but at the cost of consistency—a read of  nodes might not see the latest write until the handoff is complete.

---

### Detecting Concurrent Writes

Because any node can accept writes, different replicas may receive updates in different orders, leading to inconsistency.

#### 1. Last Write Wins (LWW)

The system attaches a timestamp to every write and picks the one with the highest timestamp as the "winner," discarding others.

* **Pros:** Achieves eventual convergence.
* **Cons:** Dangerously prone to data loss. Concurrent writes are "squashed," even if they were both successful. It is only safe if keys are immutable (e.g., using UUIDs for every write).

#### 2. The "Happens-Before" Relationship

To avoid data loss, the system must distinguish between **causal dependency** and **concurrency**.

* **Causal Dependency:** If Write B knows about Write A (or builds upon it), B is later. B should overwrite A.
* **Concurrency:** If neither write knows about the other, they are concurrent. This is a conflict that must be resolved (merged).

#### 3. Capturing Causality with Versioning

The server maintains a **version number** for every key, incrementing it with every write.

* When writing, a client must include the version number from its previous read.
* This versioning allows the server to know which previous state the write is "replacing."
* If a write arrives without a version number, or with an old one, the server keeps both values as **siblings**.

#### 4. Merging Siblings

The responsibility of resolving conflicts falls on the client (the application code).

* **Union:** For a shopping cart, taking the union of items in sibling versions is common.
* **Deletions (Tombstones):** To prevent a deleted item from "reappearing" during a merge, the system must leave a deletion marker called a **tombstone**.

---