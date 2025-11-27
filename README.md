# *Designing Data-Intensive Applications — Summary (Readable Revision Notes)*

This document provides a concise, revision-friendly overview of **Part 1** and **Part 2** of *Designing Data-Intensive Applications*.
It is formatted like a README for quick skimming and structured learning.

---

# **Overview**

This summary captures the core concepts of how modern data systems work internally and at scale.
It follows the flow of the book: from foundational single-node principles to large-scale distributed system design.

---

# **PART 1 — Foundations of Data Systems**

## **Chapter 1 — Reliability, Scalability, Maintainability**

This chapter sets the vocabulary and approach used throughout the book.

* **Reliability**: Systems should continue to function correctly despite hardware faults, software bugs, or human errors.
* **Scalability**: Systems should handle growth (requests, data, users) without performance degradation.
* **Maintainability**: Systems should be easy to operate, debug, and modify as requirements evolve.

It defines the criteria and mental model for building robust data-intensive systems.

---

## **Chapter 2 — Data Models & Query Languages**

This chapter compares multiple ways of structuring and querying data.

* Relational model & SQL (declarative querying)
* Document databases
* Graph databases
* How different models match different workloads and access patterns

The key idea: **data modeling influences everything** — performance, flexibility, and how developers think.

---

## **Chapter 3 — Storage Engines & Data Layouts**

Focuses on how databases physically store data:

* How data is laid out on disk and indexed
* Log-structured storage (LSM trees)
* Page-oriented B-Trees
* Tradeoffs: Write-optimized vs read-optimized engines

Choosing the right storage engine matters because workloads vary widely (OLTP vs analytics).

---

## **Chapter 4 — Data Encoding & Schema Evolution**

Covers how data is represented and transmitted:

* Formats for serialization: JSON, XML, Thrift, Protocol Buffers, Avro
* Forward and backward compatibility
* Handling schema changes over time
* How encoding influences performance and flexibility

---

# **PART 2 — Distributed Data Systems**

To scale beyond a single machine, systems move to distributed architectures.
This part introduces how data is **replicated**, **partitioned**, and managed across clusters.

---

## **Scaling Approaches**

### **Vertical Scaling (Scale Up)**

* Use a more powerful single machine (more CPU, RAM, disks)
* Shared-memory, fast interconnect
* **Limitations:**

  * Very expensive
  * Diminishing returns due to bottlenecks
  * Single point of failure
  * Tied to one geographic region

---

### **Shared-Disk Architecture**

* Multiple machines, each with its own CPU/RAM
* All access a **common disk array** over a fast network
* **Used in data warehousing**, but:

  * Scalability limited by lock contention
  * Hard to coordinate writers

---

### **Shared-Nothing Architecture (Horizontal Scaling / Scale Out)**

The most common modern architecture.

* Each node has **its own CPU, RAM, disk**
* Nodes coordinate **over a normal network**
* Uses cheap commodity hardware
* Supports:

  * Global distribution
  * Fault tolerance
  * Elastic scaling
* **Tradeoff**: Higher complexity and less expressive data models

Sometimes a single-threaded program beats a large cluster due to overhead, but shared-nothing remains essential for large-scale data.

---

# **Chapter 5 — Replication**

Replication means maintaining **multiple copies** of data across nodes for:

* Redundancy
* High availability
* Low-latency reads
* Higher read throughput

Replication strategies (covered in full chapter):

* Single-leader
* Multi-leader
* Leaderless
* Quorum mechanisms

---

# **Chapter 6 — Partitioning (Sharding)**

Partitioning splits a large database into smaller, independent chunks:

* Horizontal data division
* Each partition stored on different nodes
* Enables large-scale data handling
* Improves throughput by parallelism

Partitioning strategies (hash, range, etc.) and rebalancing techniques are explored in the chapter.

---

# **Chapter 7 — Transactions**

Before exploring failures in distributed systems, the book explains what **transactions** provide:

* Atomicity
* Consistency
* Isolation
* Durability

Also covers anomalies, isolation levels, and why distributed transactions are complex.

---

## **Chapters 8 & 9 — Distributed Systems Limitations**

These chapters cover fundamental impossibilities and challenges:

* Network unreliability
* Partial failures
* Time & clock issues
* CAP theorem insights
* Consensus, leader election, and the difficulty of coordination

Understanding these limitations helps engineers design realistic, fault-tolerant systems.

---