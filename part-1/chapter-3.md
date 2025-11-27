Acknowledged. I will strictly avoid using emojis in the summary.

Here is the current state of the README:

---

# Storage and Retrieval

This chapter explores the fundamental data structures and techniques databases use to efficiently **store** and **retrieve** data. It compares transactional and analytical workloads, examining how different storage architectures, like row-oriented and column-oriented systems, are optimized for specific use cases.

---

## Data Structures That Power Your Database

The simplest possible database is a **key-value store** implemented by simply **appending** records to a file, also known as a **log**.

* A `db_set` operation is highly efficient because **appending to a file is fast**.
* The system finds the most recent value for a key by scanning the file for all occurrences and taking the last one (e.g., using `tail -n 1`).
* This log-structured approach is used by many real databases internally.

### The Problem with Simple Logging

While writing is fast, the **read performance is terrible**.

* A `db_get` operation must **scan the entire file** from beginning to end to find a key.
* In algorithmic terms, lookup cost is **O(n)**, meaning performance degrades linearly as the number of records ($n$) grows.

To overcome this, a database requires an **index**.

> An **index** is an **additional structure** derived from the primary data that acts as a **signpost** to help locate data efficiently. Indexes improve **read** performance, but they introduce overhead on **writes** because the index must also be updated.

---

### Hash Indexes

A simple, viable indexing strategy, used in storage engines like **Bitcask** (default in Riak), is the **in-memory hash map**.

* **Structure:** An in-memory hash map stores the key, which maps to a **byte offset** (file location) in the data file. 
* **Write Operation:** When a new key-value pair is appended to the log file, the hash map is updated with the new offset.
* **Read Operation:** The hash map is used to find the offset, and the database **seeks** to that location to read the value.

#### Log Segments and Compaction

To prevent the append-only log from growing forever, the log is broken into **segments** (files of a certain size).

* **Compaction:** This background process throws away **duplicate keys** and retains **only the most recent value** for each key.
* **Merging:** Compaction often makes segments smaller, allowing several segments to be merged together into a new, smaller file.
    * **Immutability:** Segments are **never modified** after they are written. The merged segment is written to a new file. The old segment files are deleted only after the switch to the new merged segment is complete.
* **Lookup:** Each segment has its own hash map. To look up a key, the database checks the most recent segment's hash map first, then the second-most-recent, and so on.



#### Implementation Details (Bitcask)

| Issue | Solution |
| :--- | :--- |
| **File Format** | Use a **binary format** (length-prefixed strings) instead of CSV for faster and simpler parsing (no escaping needed). |
| **Deleting Records** | Append a special record called a **tombstone** to the log. The merging process treats the tombstone as the final value and discards all previous values for that key. |
| **Crash Recovery** | The in-memory hash map is lost on crash. To speed up rebuilding, **store a snapshot** of each segment’s hash map on disk. |
| **Partially Written** | Use **checksums** within the log files to detect and ignore corrupted or partially written records after a crash. |
| **Concurrency** | Typically, only **one writer thread** is used for sequential appending. Multiple threads can concurrently read the immutable data segments. |

#### Advantages of Append-Only Design

* **Performance:** Appending and segment merging are **sequential writes**, which are much faster than random writes, especially on magnetic HDDs and often on SSDs.
* **Simplicity:** Concurrency and crash recovery are simpler because segments are **immutable**.
* **Fragmentation:** Merging old segments avoids the problem of data files becoming fragmented over time.

#### Limitations of Hash Indexes

* **Memory Constraint:** The hash table **must fit in memory**. This makes it unsuitable for databases with a very large number of distinct keys.
* **Inefficient Range Queries:** You cannot efficiently scan a range of keys (e.g., all keys between `A` and `Z`) because the keys are not stored in any particular order in the hash map. Each key would require an individual lookup.

---

### SSTables and LSM-Trees

The **Sorted String Table (SSTable)** is a file format that improves upon the simple log segment by requiring the sequence of key-value pairs to be **sorted by key**. This simple change eliminates the limitations of Hash Indexes, especially the inability to perform range queries efficiently.

Key requirements for an SSTable segment:

1.  Key-value pairs are **sorted by key**.
2.  Each key appears **only once** within a merged segment file.

#### Advantages of SSTables

1.  **Efficient Merging:** Merging multiple segments is simple and efficient, even if the files are larger than the available memory. This process is similar to the **mergesort algorithm** .
    * Since each segment represents a time period, when a key appears in multiple segments, the value from the **most recent segment** (the one written last) is kept.

2.  **Sparse In-Memory Index:** You no longer need to keep an index of *every* key in memory.
    * The index can be **sparse** (e.g., one key for every few kilobytes).
    * To find a key, the sparse index gives you the offset of a key that appears just before the target key. Since the data is sorted, the database can jump (seek) to that offset and then **scan sequentially** until the target key is found. 

3.  **Data Compression:** Since read requests typically scan a range of keys, those key-value records can be grouped into a block and **compressed** before writing to disk.
    * The sparse index entries then point to the start of a **compressed block**.
    * Compression saves disk space and reduces I/O bandwidth usage.

#### Constructing and Maintaining SSTables (LSM-Tree)

Since incoming writes arrive in arbitrary order, the challenge is how to maintain the sorted structure efficiently. This is solved using a hybrid memory/disk approach, forming the basis of a **Log-Structured Merge-Tree (LSM-Tree)**.

1.  **Write/Insert:** When a write comes in, it is immediately added to an **in-memory balanced tree structure**, such as a Red-Black tree or AVL tree. This structure is called a **memtable**.
2.  **Memtable Flush:** When the memtable reaches a size threshold (e.g., a few megabytes), it is efficiently written out to disk as a new **SSTable file**. This is fast because the memtable is already sorted. Writes continue to a new memtable instance in the meantime.
3.  **Read Operation:** To serve a read request, the database checks:
    * **First:** The **memtable** (most recent writes).
    * **Second:** The **most recent on-disk SSTable segment**.
    * **Then:** Progressively older segments until the key is found.
4.  **Crash Recovery:** To prevent losing data in the memtable upon a crash, every write is also immediately appended to a separate, unsorted **write-ahead log** on disk. Once the memtable is successfully flushed to an SSTable, this log can be discarded.
5.  **Merging/Compaction:** A background process runs regularly to **merge and compact** the on-disk SSTable segments, combining them and discarding old, overwritten, or deleted values.

> The system described (memtable + SSTables + compaction) is known as the **Log-Structured Merge-Tree (LSM-Tree)**. This principle is used in storage engines like **LevelDB**, **RocksDB**, **Cassandra**, and **HBase**.

#### Performance Optimizations

* **Bloom Filters:** To optimize lookups for keys that **do not exist** in the database, storage engines use **Bloom filters**. A Bloom filter is a memory-efficient structure that can tell you with high probability that a key *is not* present, saving the database from checking all the SSTable segments on disk unnecessarily.
* **Compaction Strategies:** Different strategies exist for how segments are merged:
    * **Size-Tiered Compaction:** Newer, smaller SSTables are merged into older, larger ones. (Used in HBase).
    * **Leveled Compaction:** The key range is split, and older data is moved into separate "levels," allowing for more incremental compaction and less disk space usage. (Used in LevelDB/RocksDB).

The LSM-Tree architecture is effective even when the dataset exceeds available memory, supporting high write throughput (due to sequential writes) and efficient range queries (due to sorted data).

---

Here is the content for the **B-Trees** section, integrated into the README structure.

---

### B-Trees

The **B-tree** is the most widely used indexing structure, serving as the standard in almost all relational databases and many nonrelational ones. Like SSTables, B-trees keep key-value pairs **sorted by key**, supporting efficient key-value lookups and range queries.

#### Design Philosophy: Fixed-Size Pages

B-trees differ from log-structured indexes in their fundamental design:

* **Fixed Size Blocks:** B-trees break the database down into **fixed-size blocks or pages** (traditionally 4 KB). This design aligns closely with underlying disk hardware.
* **Page References:** Pages are identified by an address or location on disk. This allows one page to refer to another, constructing a tree structure.
(A four-level tree of 4 KB pages with a branching factor of 500 can store up to
256 TB)

#### Structure and Lookup

* **Tree Structure:** The index begins at the **root page**. Each page contains a set of **keys** and **references** to child pages.
* **Branching Factor:** The number of child page references in one page is the **branching factor**, which is typically several hundred in practice.
* **Lookup Process:**
    1.  Start at the **root page**.
    2.  The keys in the page define continuous key ranges. Determine which child page reference corresponds to the target key's range.
    3.  Follow the reference to the next page, which further subdivides the key range.
    4.  Repeat until a **leaf page** is reached. The leaf page contains either the final value for the key or a reference to where the value is stored.
* **Balance:** The B-tree algorithm ensures the tree remains **balanced**, meaning a tree with $n$ keys has a depth of **$O(\log n)$**. Most databases are only three or four levels deep, requiring few page reads to find a value.

#### Writing and Growing the Tree

* **Update:** To update a key, the database searches for the leaf page, changes the value in that page, and **overwrites the page** back to disk.
* **Insertion and Splitting:** To insert a new key:
    1.  Find the leaf page whose range encompasses the new key.
    2.  Add the key to that page.
    3.  If the page is full, it is **split into two half-full pages**.
    4.  The **parent page** is updated to include the new subdivision of key ranges.

#### Making B-Trees Reliable

The fundamental write operation in a B-tree is to **overwrite a page in place** (at its existing location), which is in stark contrast to the append-only nature of LSM-Trees.

* **Crash Recovery:** Overwriting multiple pages (e.g., during a page split) is dangerous, as a crash may leave the index in a corrupted, inconsistent state.
    * To ensure resilience, B-tree implementations use a **Write-Ahead Log (WAL)**, also known as a **redo log**. Every B-tree modification must first be written to this append-only log file before being applied to the tree pages.
    * After a crash, the WAL is used to restore the B-tree to a consistent state.
* **Concurrency Control:** Because pages are updated in place, concurrent access from multiple threads requires careful concurrency control, typically implemented using **latches** (lightweight locks) to protect data structures.

#### B-tree Optimizations

* **Copy-on-Write:** Instead of using a WAL and overwriting pages, some databases (like LMDB) use a **copy-on-write** scheme. Modified pages are written to a new location, and parent pages are updated to point to the new location. This aids crash recovery and concurrency.
* **Key Abbreviation:** Interior pages often do not store the entire key, only enough of it to define the **boundaries** between key ranges. This allows more keys to fit on a page, increasing the **branching factor** and decreasing the tree's depth (sometimes called a **B+ tree**).
* **Sequential Layout:** Some implementations attempt to physically lay out the leaf pages in sequential order on disk. This improves performance for **range scans**, which otherwise require a disk seek for every page read.
* **Sibling Pointers:** Pointers are often added between sibling leaf pages, allowing sequential scanning of keys without having to jump back up to the parent pages.

---

### Comparing B-Trees and LSM-Trees

Both B-trees and LSM-trees are effective indexing structures, but they have different performance profiles, largely driven by their underlying write strategies.

* **General Rule of Thumb:** **LSM-trees** are typically **faster for writes** (high write throughput), while **B-trees** are generally thought to be **faster and more predictable for reads**.
* Reads are often slower on LSM-trees because a lookup must check multiple data structures (memtable, multiple on-disk SSTables) at various stages of compaction.

#### Advantages of LSM-Trees

| Factor | LSM-Tree Advantage | Explanation |
| :--- | :--- | :--- |
| **Write Amplification (WA)** | Often lower WA, leading to higher write throughput. | WA is the ratio of one logical write resulting in multiple physical writes to disk. B-trees inherently write at least twice (WAL + Page) and often write whole pages even for small changes. LSM-trees rewrite data through compaction, but their overall WA can be lower depending on configuration. |
| **Write Throughput** | Higher throughput, especially on HDDs. | LSM-trees perform **sequential writes** of compact SSTable files, which is significantly faster than the **random overwrites** of pages used by B-trees, particularly on magnetic disks. |
| **Space Efficiency** | Better compression and lower storage overhead. | LSM-trees rewrite segments periodically, which removes fragmentation. They are not page-oriented like B-trees, which often leave unused space in pages after splits or failed row insertions. |
| **SSD Performance** | Lower WA and reduced fragmentation are still advantageous on SSDs, allowing more requests within the available I/O bandwidth. | SSD firmware often internally uses log-structured algorithms to handle random writes, lessening the impact of the storage engine's write pattern, but LSM-tree benefits persist. |

#### Downsides of LSM-Trees

| Factor | LSM-Tree Downside | Explanation |
| :--- | :--- | :--- |
| **Compaction Interference** | Can cause high latency spikes (high percentiles). | The background compaction process consumes limited disk resources, which can interfere with the performance of ongoing reads and writes. While throughput is minimally affected, **tail latencies** (response times at high percentiles) can be high and less predictable than B-trees. |
| **Compaction Lag** | Risk of growing unmerged segments under high load. | The disk's finite write bandwidth must be shared between incoming writes (logging to memtable/WAL) and background compaction. If the write rate is too high and compaction cannot keep up, the number of unmerged segments grows, slowing down reads and potentially leading to a **run-out of disk space**. |

#### B-Tree Advantages (Transactional)

* **Key Location:** Each key exists in **exactly one place** in the index.
* **Strong Transaction Semantics:** This single-location property makes B-trees attractive for databases offering strong transactional semantics. Features like **locking ranges of keys** (for transaction isolation) can be directly attached to the B-tree structure, which is more complicated when keys may exist in multiple segments, as in LSM-trees.

Due to their maturity and predictable performance, B-trees remain a foundational technology. However, log-structured indexes are increasingly popular in new datastores due to their superior write throughput. The best choice depends entirely on the specific application's workload, making empirical testing essential.

---

### Other Indexing Structures

The key-value indexes discussed so far are analogous to a **primary key index**. Databases also rely heavily on **secondary indexes** to efficiently query data on non-primary-key columns, which are crucial for joins.

#### Secondary Indexes

A secondary index is built on columns that are **not necessarily unique**. This non-uniqueness can be handled in two ways:

1.  **List of Identifiers:** The index value is a list of matching row identifiers (analogous to a **postings list** in full-text search).
2.  **Unique Key:** The key is made unique by **appending the row identifier** to it.

Both B-trees and LSM-trees can be used to implement secondary indexes.

#### Storing Values within the Index

The index's **value** can be one of two things:

1.  **Reference to Data (Nonclustered Index):** The index value is a reference (e.g., a byte offset) to the row's location in a **heap file**.
    * A **heap file** stores the data itself in no particular order. This method is common because it avoids data duplication: all secondary indexes simply point to the single copy of the data in the heap.
    * Updates where the new value is larger may require moving the row and updating all index references, or using a **forwarding pointer** at the old location.
2.  **The Actual Data (Clustered Index):** The indexed row data is stored directly within the index structure itself.
    * This avoids the "extra hop" from index to heap file, improving **read performance**.
    * **Example:** In MySQL's InnoDB, the primary key is always a clustered index, and secondary indexes reference the primary key.
3.  **Covering Index (Index with Included Columns):** A compromise where the index stores only **some** of the table's columns alongside the key. If a query only needs the indexed or included columns, it can be answered entirely by the index (**covering the query**), avoiding a trip to the main data.

> **Trade-off:** Clustered/covering indexes speed up reads but require more storage, introduce data duplication, and add overhead on writes (since multiple structures must be updated).

#### Multi-Column Indexes

These indexes allow queries to search multiple columns simultaneously.

* **Concatenated Index:** The most common type, where multiple fields are simply combined into a single key (e.g., `(lastname, firstname)`). This allows efficient searching on the combined key or on the leading column(s), but is useless for searching on non-leading columns (e.g., only `firstname`).
* **Multi-dimensional Indexes:** These are specialized structures, often used for **geospatial data** (latitude, longitude) but applicable to any multi-dimensional range query (e.g., searching for products by `(color, size)` or weather data by `(date, temperature)`).
    * A standard B-tree cannot efficiently answer a query on two independent ranges simultaneously.
    * Common solutions include using **space-filling curves** to map 2D location to a single number for a regular B-tree, or using specialized structures like **R-trees**. 

#### Full-Text Search and Fuzzy Indexes

These structures are designed for non-exact queries, such as searching for misspelled words.

* **Full-Text Search:** Engines like Lucene use techniques to find synonyms, ignore grammatical variations, and search for proximity of words.
    * Lucene implements its term dictionary using an **SSTable-like structure**. Its in-memory index is a **finite state automaton** (similar to a trie) over the characters in the keys.
    * This automaton can be converted into a **Levenshtein automaton** to efficiently search for words within a specified **edit distance** (a measure of similarity based on insertions, deletions, or substitutions).

#### Keeping Everything in Memory

In-memory databases leverage falling RAM costs to store the entire working dataset in memory, sacrificing disk's durability for simplicity and performance.

* **Durability:** Achieved by writing a log of changes to disk, writing periodic snapshots, or replicating the state to other machines.
* **Performance Advantage:** The primary benefit is not avoiding disk reads (since OS caches often handle that anyway), but **avoiding the overheads** of translating and encoding in-memory data structures into a format suitable for on-disk management.
* **Anti-caching:** A technique where an in-memory database can support datasets larger than RAM by **evicting the least recently used records** to disk when memory is scarce (similar to OS swapping, but at the record level). This approach still generally requires the index itself to fit entirely in memory.

---

## Transaction Processing or Analytics?

Database usage can be broadly divided into two main categories, based on the access patterns they are optimized for:

* **Online Transaction Processing (OLTP):** Characterized by frequent, low-latency reads and writes of a small number of records (e.g., fetching a record by a primary key). These are used in interactive applications (web apps, checkouts).
* **Online Analytic Processing (OLAP):** Characterized by queries that scan over a huge number of records, reading only a few columns, and calculating aggregate statistics (e.g., `SUM`, `COUNT`, `AVG`). These are used by business analysts for decision support (business intelligence).

| Property | Transaction Processing Systems (OLTP) | Analytic Systems (OLAP) |
| :--- | :--- | :--- |
| **Main Read Pattern** | Small number of records per query, fetched by key. | Aggregate over a large number of records. |
| **Main Write Pattern** | Random-access, low-latency writes from user input. | Bulk import (ETL) or event stream. |
| **Primary User** | End user/customer, via web application. | Internal analyst, for decision support. |
| **Data Representation** | Latest state of data (current point in time). | History of events that happened over time. |
| **Dataset Size** | Gigabytes to terabytes. | Terabytes to petabytes. |

---

### Data Warehousing

Since OLTP systems are critical for business operations and require high availability and low latency, ad hoc analytic queries (which are often expensive and long-running) are typically **prohibited** on the primary OLTP database, as they could harm performance.

A **data warehouse** is a separate, dedicated database optimized specifically for analytic access patterns.

* **Purpose:** It contains a read-only copy of data from all the company's various OLTP systems. Analysts can query this warehouse without affecting the performance of live transactional operations.
* **ETL Process:** Data is moved into the warehouse using an **Extract–Transform–Load (ETL)** process:
    1.  **Extract:** Data is pulled from OLTP databases.
    2.  **Transform:** Data is cleaned up and converted into a schema optimized for analysis.
    3.  **Load:** Data is imported into the data warehouse.

The divergence between OLTP databases (optimized for random-access writes and indexed lookups) and data warehouses (optimized for large-scale aggregate scans) has led many database vendors to specialize their products for one workload or the other.

---

### Stars and Snowflakes: Schemas for Analytics

The data model for most data warehouses is relational, but follows a specific, formulaic structure known as a **star schema** (or dimensional modeling).

* **Fact Table:** This is at the center of the schema (e.g., `fact_sales`). Each row represents a single event that occurred (e.g., a customer purchase or a page view).
    * Fact tables capture events as individual records for maximum analysis flexibility and can become **extremely large** (petabytes).
    * They contain **attributes** (e.g., price, cost) and **foreign keys** to dimension tables.
* **Dimension Tables:** These tables surround the fact table and represent the context (the *who, what, where, when, how, and why*) of the event.
    * **Example:** A `dim_product` table contains all the metadata about a product (brand, category, size).
    * The foreign keys in the fact table link to the dimension tables to retrieve descriptive attributes for any event.

* **Snowflake Schema:** A variation where dimensions are further **normalized** by being broken down into subdimensions (e.g., the `dim_product` table references separate tables for brands and categories). Star schemas are generally preferred for their simplicity for analysts.
* **Table Width:** Tables in data warehouses are often very **wide** (e.g., fact tables with over 100 columns and dimension tables containing all relevant analysis metadata).

---

### Column-Oriented Storage

Traditional **Online Transaction Processing (OLTP)** databases use **row-oriented storage**, where all values for a single row are stored contiguously on disk.

However, in **Online Analytical Processing (OLAP)**, queries often access only a few columns of a very wide fact table (e.g., 4 or 5 out of 100+ columns).

* **Problem with Row-Oriented Storage:** To process an analytical query, a row-oriented storage engine must load the **entire row** from disk into memory, parse it, and then discard the unnecessary columns. This wastes significant I/O bandwidth.

**Column-Oriented Storage** solves this by storing all the values from **each column together**, separate from other columns. 

* **Efficiency:** A query only needs to read and parse the specific column files it requires.
* **Row Reconstruction:** The column files must store rows in the same order. To reconstruct the $k^{th}$ row, the system takes the $k^{th}$ entry from each required column file.

#### Column Compression

Column-oriented storage is highly amenable to data compression, which further reduces disk I/O requirements.

* **Bitmap Encoding:** A technique that is very effective when a column has a small number of **distinct values** relative to the number of rows (e.g., 100,000 distinct products in billions of transactions).
    1.  The column is converted into $n$ separate **bitmaps** (one for each distinct value).
    2.  Each bitmap has one bit per row: $1$ if the row has that value, $0$ otherwise.
    3.  Sparse bitmaps (those with mostly $0$s) can be further compressed using **run-length encoding**.
* **Query Optimization:** Bitmap indexes allow for very fast processing of conditions:
    * `WHERE product_sk IN (30, 68, 69)`: Load the three bitmaps and calculate the **bitwise OR**.
    * `WHERE product_sk = 31 AND store_sk = 3`: Load the two bitmaps and calculate the **bitwise AND**. This works because the $k^{th}$ bit in all columns corresponds to the same row.

#### Memory Bandwidth and Vectorized Processing

Columnar storage also improves **CPU efficiency** through **vectorized processing**.

* **Vectorized Processing:** The query engine works on a **chunk of compressed column data** that fits entirely in the CPU's L1 cache. The CPU iterates through this data in a tight loop, avoiding the overhead of function calls and branch mispredictions common in row-by-row processing.
* **Benefit:** Compression allows more rows to fit into the L1 cache, maximizing the use of CPU and memory bandwidth.

#### Sort Order in Column Storage

While rows in a column store can be in any order, choosing a specific sort order for the entire row set (even though the data is stored by column) offers performance benefits:

* **Query Speed:** Sorting by a frequently queried column (e.g., `date_key`) allows the query optimizer to scan only the necessary range of rows (e.g., only the last month's data).
* **Compression Improvement:** Sorting causes the primary sort key column to have long sequences of repeated values. A simple **run-length encoding** on this column can achieve massive compression, as the column is not randomized.
* **Multiple Sort Orders:** Some systems (e.g., Vertica, inspired by C-Store) store the **same data sorted in several different ways** across redundant copies. The query planner then chooses the version best suited for the specific query, similar in goal to having multiple secondary indexes in a row store.

#### Writing to Column-Oriented Storage

Columnar storage, compression, and sorting are optimized for reads but complicate writes.

* **Challenge:** Traditional update-in-place (like B-trees) is impossible because inserting a row in the middle of a sorted, compressed table would require rewriting all column files.
* **Solution: LSM-Trees:** Column-oriented storage engines often use the **LSM-tree principle** to handle writes:
    1.  Writes go to an in-memory store (memtable).
    2.  When enough writes accumulate, they are merged with the on-disk column files and written out in **bulk** to new files.
    3.  Queries combine the recent writes in memory with the data on disk, ensuring that all modifications are immediately visible to the analyst.

---

### Aggregation: Data Cubes and Materialized Views

Analytical queries frequently involve aggregate functions (`SUM`, `COUNT`, `AVG`). If the same aggregates are computed repeatedly, the results can be cached to avoid reprocessing the raw data.

* **Materialized View:** A table-like object that is an **actual copy** of the results of an underlying query, stored on disk.
    * Unlike a standard (virtual) view, a materialized view must be explicitly updated when the underlying data changes, making writes more expensive. They are primarily used in read-heavy data warehouses.
* **Data Cube (OLAP Cube):** A common special case of a materialized view. It is a **grid of aggregates** grouped by different dimensions (e.g., date, product, store).
    * **Precomputation:** Each cell in the hypercube contains the precomputed aggregate (e.g., total sales) for a specific combination of dimension values.
    * **Advantage:** Queries for summaries along any dimension (e.g., total sales by store regardless of product) are extremely fast, as the results are precomputed.
* **Disadvantage:** Data cubes lack the flexibility of querying raw data. They cannot answer questions that involve attributes *not* included as dimensions in the cube (e.g., sales proportion for items over $100$).

Most data warehouses prioritize keeping **raw data** for maximum flexibility and use materialized aggregates only as a performance boost for common queries.

---

## Summary

This chapter explored the fundamental data structures and storage architectures used in modern databases:

* **Indexing Structures:** Databases rely on indexes to efficiently find data.
    * **Hash Indexes:** Fast for exact key lookups but must fit in memory and cannot support range queries.
    * **B-Trees:** The standard for most transactional databases. They use fixed-size pages, overwrite data in place, and require a WAL for crash recovery. They offer good, predictable performance.
    * **LSM-Trees:** The core of modern log-structured storage (SSTables). They are append-only, use compaction/merging, and offer higher write throughput and better compression but can have less predictable read latencies due to background operations.
* **Workloads:** Databases are optimized for either **Online Transaction Processing (OLTP)** (low-latency, random access) or **Online Analytical Processing (OLAP)** (large-scale aggregation).
* **Data Warehousing:** Separates analytics from transactional systems using an **ETL** process, often employing a **star schema** for data modeling.
* **Column-Oriented Storage:** Optimized for OLAP by storing all values of a column together. This reduces I/O by only reading necessary columns and enables highly effective data compression and **vectorized processing**.
