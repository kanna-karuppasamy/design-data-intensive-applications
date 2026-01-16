This chapter explores **Transactions**, a crucial abstraction that simplifies the "harsh reality" of distributed systems. In a world where networks fail, hardware crashes, and race conditions are rampant, transactions provide a safety net for developers.

---

# Chapter 7: Transactions

A transaction is a way for an application to group several reads and writes into a **logical unit**. Conceptually, the entire unit is treated as a single operation: either it succeeds (**commit**) or it fails (**abort/rollback**).

## The Meaning of ACID

The safety guarantees of transactions are summarized by the acronym **ACID**. While these terms are widely used, their definitions are often slippery and vary between database vendors.

### Atomicity

Contrary to its meaning in multi-threaded programming (where it relates to isolation), ACID atomicity is about **abortability**.

* **The Guarantee:** If an error occurs halfway through a sequence of writes, the transaction is aborted and all prior writes are discarded.
* **The Benefit:** The application doesn't have to worry about "partial failure." If a transaction fails, it can be safely retried without leaving the data in a corrupted state.

### Consistency

In ACID, "Consistency" is an outlier—it is actually a property of the **application**, not the database.

* **The Definition:** The database must transition from one valid state to another, maintaining "invariants" (e.g., account balances must sum to zero).
* **The Reality:** The database can enforce simple constraints (foreign keys, uniqueness), but general consistency is the programmer's responsibility.

### Isolation

Most databases are accessed by multiple clients at once. If they access the same records, they trigger **race conditions**.

* **The Guarantee:** Concurrently executing transactions are isolated from each other.
* **The Ideal:** **Serializability**—the database ensures the result is the same as if transactions ran one after another.
* **The Reality:** Because serializability is slow, most databases use weaker levels like "Read Committed" or "Snapshot Isolation."

### Durability

Durability is the promise that once a transaction commits, the data will not be lost.

* **Single-node:** Data is written to non-volatile storage (HDD/SSD) and a write-ahead log.
* **Replicated:** Data is successfully copied to a specific number of nodes.

---

## Single-Object vs. Multi-Object Operations

In ACID, atomicity and isolation are most critical when modifying multiple records simultaneously.

### The Need for Multi-Object Transactions

Many use cases require updates to stay in sync across different objects:

1. **Foreign Keys:** Ensuring a reference in one table exists in another.
2. **Denormalization:** When an "unread count" must be incremented in one document while a new email is added to another.
3. **Secondary Indexes:** Updating the search index alongside the actual data record.

### Single-Object "Atomicity"

Almost all storage engines provide atomicity and isolation for a **single object** (e.g., updating one JSON document). This prevents issues like reading a half-written document. Some call this "lightweight ACID," but it is not a "transaction" in the traditional sense because it doesn't coordinate multiple objects.

---

## Handling Aborts and Retries

The defining feature of a transaction is that it can be **aborted and safely retried**. This is the database's way of saying, "I can't guarantee safety right now, so I'm undoing everything so you can try again."

**Challenges with retries:**

* **Network Failure:** If the commit succeeded but the acknowledgement failed, retrying might perform the action twice (requires **idempotence**).
* **Overload:** Retrying too aggressively can worsen a database bottleneck (requires **exponential backoff**).
* **Side Effects:** If the transaction triggers an email, you don't want that email sent every time the transaction retries.

---

### Next Step

We've established the "why" and "what" of transactions. The next logical step is to dive into the "how."

**Would you like to move on to Weak Isolation Levels?** We will explore:

* **Read Committed** (preventing dirty reads/writes).
* **Snapshot Isolation** (using MVCC to prevent read skew).
* **Common Race Conditions** (Lost Updates, Write Skew, and Phantoms).This chapter explores **Transactions**, a crucial abstraction that simplifies the "harsh reality" of distributed systems. In a world where networks fail, hardware crashes, and race conditions are rampant, transactions provide a safety net for developers.

---

# Chapter 7: Transactions

A transaction is a way for an application to group several reads and writes into a **logical unit**. Conceptually, the entire unit is treated as a single operation: either it succeeds (**commit**) or it fails (**abort/rollback**).

## The Meaning of ACID

The safety guarantees of transactions are summarized by the acronym **ACID**. While these terms are widely used, their definitions are often slippery and vary between database vendors.

### Atomicity

Contrary to its meaning in multi-threaded programming (where it relates to isolation), ACID atomicity is about **abortability**.

* **The Guarantee:** If an error occurs halfway through a sequence of writes, the transaction is aborted and all prior writes are discarded.
* **The Benefit:** The application doesn't have to worry about "partial failure." If a transaction fails, it can be safely retried without leaving the data in a corrupted state.

### Consistency

In ACID, "Consistency" is an outlier—it is actually a property of the **application**, not the database.

* **The Definition:** The database must transition from one valid state to another, maintaining "invariants" (e.g., account balances must sum to zero).
* **The Reality:** The database can enforce simple constraints (foreign keys, uniqueness), but general consistency is the programmer's responsibility.

### Isolation

Most databases are accessed by multiple clients at once. If they access the same records, they trigger **race conditions**.

* **The Guarantee:** Concurrently executing transactions are isolated from each other.
* **The Ideal:** **Serializability**—the database ensures the result is the same as if transactions ran one after another.
* **The Reality:** Because serializability is slow, most databases use weaker levels like "Read Committed" or "Snapshot Isolation."

### Durability

Durability is the promise that once a transaction commits, the data will not be lost.

* **Single-node:** Data is written to non-volatile storage (HDD/SSD) and a write-ahead log.
* **Replicated:** Data is successfully copied to a specific number of nodes.

---

## Single-Object vs. Multi-Object Operations

In ACID, atomicity and isolation are most critical when modifying multiple records simultaneously.

### The Need for Multi-Object Transactions

Many use cases require updates to stay in sync across different objects:

1. **Foreign Keys:** Ensuring a reference in one table exists in another.
2. **Denormalization:** When an "unread count" must be incremented in one document while a new email is added to another.
3. **Secondary Indexes:** Updating the search index alongside the actual data record.

### Single-Object "Atomicity"

Almost all storage engines provide atomicity and isolation for a **single object** (e.g., updating one JSON document). This prevents issues like reading a half-written document. Some call this "lightweight ACID," but it is not a "transaction" in the traditional sense because it doesn't coordinate multiple objects.

---

## Handling Aborts and Retries

The defining feature of a transaction is that it can be **aborted and safely retried**. This is the database's way of saying, "I can't guarantee safety right now, so I'm undoing everything so you can try again."

**Challenges with retries:**

* **Network Failure:** If the commit succeeded but the acknowledgement failed, retrying might perform the action twice (requires **idempotence**).
* **Overload:** Retrying too aggressively can worsen a database bottleneck (requires **exponential backoff**).
* **Side Effects:** If the transaction triggers an email, you don't want that email sent every time the transaction retries.

---

In an ideal world, we would use **Serializable Isolation**, where the database ensures transactions behave as if they ran one after another. However, due to the high performance costs of serializability, most applications rely on **Weak Isolation Levels**.

These levels protect against some race conditions but leave others exposed. Understanding these trade-offs is essential for building reliable systems, as timing-related bugs are notoriously difficult to reproduce in testing.

---

## Read Committed

This is the most basic level of isolation and is the default in databases like PostgreSQL, Oracle, and SQL Server. It provides two core guarantees:

### 1. No Dirty Reads

You only see data that has already been **committed**. If Transaction A changes a value but hasn't committed yet, Transaction B will see the old value.

* **Why it matters:** Prevents an application from acting on data that might be rolled back (aborted) or seeing a "half-finished" state (e.g., an email is visible but the unread counter hasn't updated).

### 2. No Dirty Writes

You can only overwrite data that has been **committed**. If Transaction A is currently writing to an object, Transaction B must wait until A finishes before it can begin its write.

* **Implementation:** Databases typically use **row-level locks**.
* **Why it matters:** Prevents "mingled" updates where, for example, two people try to buy a car and one gets the listing while the other gets the invoice.

---

## Snapshot Isolation (Repeatable Read)

While *Read Committed* handles simple cases, it fails to prevent **Read Skew** (nonrepeatable reads).

### The Read Skew Problem

If Alice transfers $100 from Account A to Account B, a background backup process might read Account A *after* the deduction but read Account B *before* the credit arrives. To the backup, $100 appears to have vanished.

### Multi-Version Concurrency Control (MVCC)

Snapshot Isolation solves this by ensuring a transaction sees a **consistent snapshot** of the database as it existed at the moment the transaction started.

* **Key Principle:** Readers never block writers, and writers never block readers.
* **Implementation:** The database keeps multiple versions of an object. Each row is tagged with a `created_by` and `deleted_by` transaction ID (txid).

#### Visibility Rules

For a transaction to see an object:

1. The transaction that created the object must have committed **before** the current transaction started.
2. The object must not be marked for deletion (or the deletion transaction hasn't committed yet).

---

## Comparison of Race Conditions

| Race Condition | Description | Prevented by Read Committed? | Prevented by Snapshot Isolation? |
| --- | --- | --- | --- |
| **Dirty Read** | Reading uncommitted data. | ✅ Yes | ✅ Yes |
| **Dirty Write** | Overwriting uncommitted data. | ✅ Yes | ✅ Yes |
| **Read Skew** | Seeing different data at different times. | ❌ No | ✅ Yes |
| **Lost Update** | Two writes "clobber" each other. | ❌ No | ❌ No* |

**Note: Some implementations of Snapshot Isolation can detect lost updates, but it is not guaranteed by the definition itself.*

---

## The Naming Confusion

There is significant "terminological mess" regarding these levels:

* **Oracle** calls its Snapshot Isolation "Serializable."
* **PostgreSQL** and **MySQL** call their Snapshot Isolation "Repeatable Read."
* **DB2** uses "Repeatable Read" to mean true Serializability.

Because the SQL standard's original definitions were ambiguous, you must check your specific database documentation to know exactly what guarantees you are getting.

While **Read Committed** and **Snapshot Isolation** protect against many reading anomalies, they are often insufficient when multiple clients attempt to write to the same or related data simultaneously. This leads to two of the most dangerous race conditions in database systems: **Lost Updates** and **Write Skew**.

---

## 1. The Lost Update Problem

A lost update occurs during a **read-modify-write** cycle. If two transactions read a value, modify it locally, and write it back, the second write will "clobber" the first, causing the first modification to vanish.

### Solutions for Lost Updates

| Method | Description | Example/Note |
| --- | --- | --- |
| **Atomic Writes** | The database performs the update in a single operation, removing the application's need to read first. | `UPDATE counters SET value = value + 1 WHERE key = 'foo';` |
| **Explicit Locking** | The application forces a lock on the row during the initial read so no one else can touch it. | `SELECT ... FOR UPDATE;` |
| **Automatic Detection** | The database tracks conflicts and aborts one transaction if it would result in a lost update. | Supported by PostgreSQL (Repeatable Read), but **not** MySQL. |
| **Compare-and-Set** | The write only succeeds if the value hasn't changed since it was last read. | `UPDATE ... WHERE content = 'old content';` |

---

## 2. Write Skew and Phantoms

**Write Skew** is a more subtle race condition. It occurs when two transactions read the same data, but update **different** objects in a way that violates a business requirement.

### The "On-Call Doctors" Example

Imagine a hospital requires at least one doctor to be on call.

1. Alice and Bob are both on call (Total = 2).
2. Alice starts a transaction, sees there are 2 doctors, and goes off call.
3. Simultaneously, Bob starts a transaction, sees 2 doctors (because Alice hasn't committed), and goes off call.
4. Both commit. **Result:** Zero doctors are on call.

### Phantoms

A **phantom** occurs when a write in one transaction changes the result of a search query in another.

* In the doctor example, the check was for the *presence* of rows.
* In other cases, like a meeting room booking, the check is for the *absence* of rows (e.g., "Are there any overlapping meetings?"). Since the meeting doesn't exist yet, there is no row to lock.

---

## Summary of Write-Write Conflicts

| Conflict | Different Objects? | Common Solution |
| --- | --- | --- |
| **Dirty Write** | No (Same object) | Row-level locks (built into Read Committed). |
| **Lost Update** | No (Same object) | Atomic operations, `FOR UPDATE`, or detection. |
| **Write Skew** | **Yes** (Different objects) | **Serializability** or materializing conflicts. |

### Materializing Conflicts

If you cannot use serializable isolation, you can "materialize" a phantom by creating a physical table of locks. For a booking system, you might create a `time_slots` table. Instead of checking for overlapping meetings, you lock the specific "slot" rows. This turns a phantom into a concrete lock conflict.

---

## Statistics: Database Isolation Defaults

Understanding the risk in your stack is vital. Here are the default isolation levels for major RDBMS providers:

| Database | Default Isolation Level | Prevents Lost Updates? | Prevents Write Skew? |
| --- | --- | --- | --- |
| **Oracle** | Read Committed | No | No |
| **PostgreSQL** | Read Committed | No | No |
| **SQL Server** | Read Committed | No | No |
| **MySQL (InnoDB)** | Repeatable Read | No | No |

*Note: While MySQL's "Repeatable Read" sounds stronger, it famously does not detect lost updates or prevent write skew.*

We have now reached the limits of weak isolation. To truly solve Write Skew and Phantoms without manual "hacks" like materializing conflicts, we need the strongest guarantee.

To solve the persistent issues of **Write Skew** and **Phantoms**, researchers have long advocated for the strongest level of isolation: **Serializability**. It guarantees that even if transactions run in parallel, the end result is exactly the same as if they had run one at a time (serially).

There are three main ways modern databases implement this level of safety.

---

## 1. Actual Serial Execution

The simplest way to avoid concurrency issues is to remove the concurrency entirely. By executing only one transaction at a time on a single thread, you sidestep the need for complex locking or conflict detection.

### Why it works now (but didn't before):

Historically, single-threading was considered too slow because of disk I/O. Two shifts made it viable:

* **In-Memory Storage:** RAM is now cheap enough to hold active datasets. If a transaction doesn't have to wait for a disk, it executes in microseconds.
* **OLTP focus:** Most "Online Transaction Processing" tasks are short. Long-running analytical queries are offloaded to consistent snapshots using MVCC, leaving the serial thread free for fast writes.

### The Shift to Stored Procedures

In an "interactive" transaction, the database waits for the application to send the next command over the network. In a single-threaded system, this idle time would kill performance.

* **The Solution:** The application sends the entire transaction logic to the database as a **Stored Procedure**.
* **Modern Languages:** While old stored procedures used "archaic" languages like PL/SQL, modern systems use Java (VoltDB), Lua (Redis), or Clojure (Datomic).

### Limitations of Serial Execution:

1. **CPU Bottleneck:** Throughput is limited to a single CPU core.
2. **Partitioning Pains:** You can scale by partitioning data across cores, but **cross-partition transactions** require heavy coordination and are orders of magnitude slower.
3. **Non-deterministic code:** Transactions must produce the same result every time they run (critical for replication).

---

## 2. Two-Phase Locking (2PL)

For decades, this was the gold standard for serializability. It is much stronger than the simple locking used in Read Committed.

* **The Rule:** If Transaction A wants to read an object and Transaction B wants to write to it, one must wait for the other. Unlike Snapshot Isolation, **writers block readers and readers block writers.**
* **The "Two Phases":** 1.  **Expansion:** Acquiring locks during the transaction.
2.  **Shrinking:** Releasing all locks only at the end (Commit or Abort).
* **Downside:** High latency and risk of **deadlocks**, where two transactions are waiting for each other’s locks.

---

## 3. Serializable Snapshot Isolation (SSI)

This is a newer, "optimistic" approach (used in PostgreSQL since version 9.1).

* **The Logic:** Instead of blocking, it lets transactions proceed as if everything is fine. When a transaction tries to commit, the database checks if any "isolation violations" (like write skew) occurred.
* **The Result:** If it detects a conflict, the transaction is aborted and must be retried.
* **Pros:** It performs much better than 2PL in systems with low contention because it avoids unnecessary blocking.

---

### Comparison Summary

| Method | Approach | Best For | Used In |
| --- | --- | --- | --- |
| **Serial Execution** | Pessimistic (Single Thread) | In-memory, fast OLTP | Redis, VoltDB |
| **Two-Phase Locking** | Pessimistic (Blocking) | Heavy write contention | SQL Server, DB2 |
| **SSI** | Optimistic (Detection) | Read-heavy workloads | PostgreSQL, FoundationDB |

### Next Step

We have covered how to achieve safety on a **single node**. However, modern applications often need to scale across multiple machines.