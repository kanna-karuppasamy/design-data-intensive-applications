# **Data Models and Query Languages — Summary**

## **Why Data Models Matter**

Data models shape:

* How we understand the real world
* How we design software
* How systems perform, scale, and evolve

Choosing the right model influences **efficiency, scalability, and maintainability**.

---

## **Layers of Data Modeling**

Software is built in layers. Each layer hides the complexity of the one below it.

| Layer                          | What It Models                      | Examples                                     |
| ------------------------------ | ----------------------------------- | -------------------------------------------- |
| **1. Application Level**       | Real-world objects                  | Classes like `User`, `Order`, `Organisation` |
| **2. Storage Model Level**     | App objects → Standard formats      | JSON, XML, tables, graph nodes               |
| **3. Database Internal Level** | Storage model → Physical structures | B-trees, LSM trees, heap files               |
| **4. Hardware Level**          | Bytes → Physical representation     | RAM cells, SSD pages, disk blocks            |

---

# **Relational vs Document Data Models**

## **Relational Model (SQL)**

* Dominant for 30+ years. It was introduced by Edgar Codd in 1970 to hide low-level storage complexity and make data easier to work with.
* Introduced to hide low-level details
* Works extremely well for structured, relational, interconnected data
* Performs best with **many-to-one** and **many-to-many** relationships

## **Document Model (JSON)**

* Stores entire objects as nested documents
* Matches application structures more naturally
* Great for hierarchical data that’s read in one go

# **Why NoSQL Became Popular**

Several major forces pushed developers toward NoSQL systems:

1. **Scalability Limits of Traditional RDBMS**
    - Modern applications began handling **huge datasets and very high write throughput**.
    - Scaling relational databases vertically (bigger servers) became expensive and reached practical limits.
    - NoSQL systems **scaled horizontally** across many machines, which worked better for web-scale growth.
2. **Shift Toward Open Source**
    - Many organizations preferred **free and open source** data infrastructure over costly commercial SQL databases.
    - This aligned with the rise of startups and cloud-native development.
3. **New Query and Data Access Patterns**
    - Some applications (e.g., real-time analytics, search, recommendation graphs) needed operations **not supported efficiently** by the relational model.
    - NoSQL databases introduced specialized approaches (document stores, key-value stores, wide-column databases, graph databases).
4. **Flexibility in Data Modeling**
    - Relational schemas are **strict and require upfront design**.
    - Fast-changing applications needed **schema flexibility**.
    - NoSQL allowed storing data in **dynamic structures**, often without predefined schemas.

The key idea is **polyglot persistence**:
Different applications need different storage models, so relational and NoSQL databases are likely to coexist rather than one replacing the other.

# **Object–Relational Mismatch (Impedance Mismatch)**

Modern applications are written using **object-oriented programming**, but most data is stored in **relational databases** (tables). This causes a **mismatch** between how data is represented in code (objects) and how it is stored (rows/columns). This difficulty is called the **impedance mismatch**.
ORM tools like **Hibernate** and **ActiveRecord** try to bridge this gap, but **they cannot eliminate it completely**.

### Example (LinkedIn-like Profile)

**Relational:**

* User → separate tables (positions, education, contacts)
* Requires joins / multiple queries

Tools like Hibernate or GORM (for Go) exist specifically to write the SELECT * FROM ... JOIN ... code for you. However, they can't fix the performance cost of the joins—they just hide the "ugly" SQL from your view.

**Document:**

```json
{
  "user_id": 251,
  "first_name": "Bill",
  "positions": [
    { "job_title": "Co-chair", "organization": "Bill & Melinda Gates Foundation" }
  ]
}
```
Benefits:

- **Everything is stored together** → one query to fetch.
- The natural **tree structure** of data is preserved.
- Often feels more intuitive in application code → **less mismatch**.

Trade-offs:

- JSON can lack **schema guarantees** (may lead to messy or inconsistent data).
- Some databases cannot **query inside JSON** efficiently (varies by database).

| Relational (SQL)                              | Document (JSON)                           |
| ----------------------------------------------| ------------------------------------------|
| Splits related data across tables             | All related data stored in one document   |
| Requires joins/multiple queries               | One query usually enough                  |
| Schema is strict and enforced                 | Schema flexibility (both benefit and risk)|
| Works well for structured, relational data    | Works well for hierarchical, nested data  |


### Many-to-One and Many-to-Many Relationships

In the résumé example, fields like `region_id` and `industry_id` are stored as **IDs**, not text like "Greater Seattle Area" or "Philanthropy". This is done to **avoid duplication of meaningful text** and to ensure **consistency** across the database.

### **Benefits of Using IDs**

1. **Consistent Naming**
    
    Everyone references the same standardized term, preventing spelling or wording differences.
    
2. **Avoid Ambiguity**
    
    Cities or regions with the same name are not confused.
    
3. **Easy Updates**
    
    If a region name changes, update it in *one place*, and all references remain correct.
    
4. **Localization**
    
    The same region can be shown in different languages depending on the viewer.
    
5. **Better Search and Classification**
    
    The database can understand relationships like “Seattle is in Washington,” improving search accuracy.
    

### **Normalization and Duplication**

- Using IDs avoids duplicating human-meaningful text across many records.
- Human-readable text may change over time, while IDs remain stable.
- Removing duplication is the **core idea of normalization**.
- If data is duplicated, updating it everywhere becomes slow and error-prone.

### **The Issue in Document Databases**

- Relational databases handle **many-to-one** and **many-to-many** relationships easily using **joins**.
- Document databases (like MongoDB) work best with **one-to-many tree structures** inside a single document.
- They have **weak or no join support**, so cross-references require:
    - Extra queries in application code, or
    - Storing duplicated data (which risks inconsistency).

### **When Data Becomes More Connected**

As new features are added, data often becomes more interlinked:

- Organizations and schools may become their own entities.
- Users may write recommendations for each other.
- These features introduce **many-to-many relationships**.

This makes the data model **less tree-like**, which fits **relational databases better** than document databases.

| Situation | Best Model |
| --- | --- |
| Data is hierarchical and mostly self-contained | **Document (JSON)** |
| Data has many references across entities (many-to-one or many-to-many) | **Relational (SQL)** |

### **Are Document Databases Repeating History?**

This debate is not new. In the 1970s, databases used a **hierarchical model**, very similar to how **JSON document databases** store nested data today. For example, IBM’s **IMS** stored data as **trees of records**, just like a JSON document.

### **What Worked Well**

- One-to-many relationships fit naturally (a user with multiple jobs, addresses, etc.).
- Data stored together was fast to read.

### **The Problem**

When data needed **many-to-one** or **many-to-many** relationships (for example, multiple users linked to one region or company), the hierarchical model made things **difficult**:

- Either duplicate the data everywhere (risk inconsistent updates), or
- Manually follow reference links, which was painful and error-prone.

This is exactly the situation today when document databases force developers to:

- Duplicate data (denormalize), or
- Fetch related documents manually in application code.

### **Relational vs Document Databases Today**

When comparing relational (SQL) and document (NoSQL) databases, the core difference lies in how **data is structured and queried**.

### **When Document Databases Work Well**

Use a **document model** when your data naturally looks like a **tree** (one-to-many) and you usually need to load the entire object together. Example: a user profile with jobs, education, and contact info.

Advantages:

- **Flexible schema** (schema-on-read): you can add new fields anytime without migrations.
- **Performance locality**: one document stored together makes reads faster if you need the whole object.
- **Matches how many applications actually structure data in memory.**

Limitation:

- **Poor support for joins.** If your data becomes interconnected (many-to-many), you either:
    - Denormalize and duplicate data (risk inconsistency), or
    - Do manual joins in application code (performance cost and complexity).

### **When Relational Databases Are Better**

Use **relational (SQL)** when you have:

- **Many-to-one** or **many-to-many** relationships.
- Data that becomes more interconnected as features grow.

Advantages:

- **Joins are first class** and highly optimized by the database.
- Data remains **consistent** across references.
- Evolving queries over time is simpler.

Limitation:

- Can feel **cumbersome** to break document-like data into multiple tables.

### **Declarative Querying (Relational Model → SQL)**

Declarative querying means:

* You tell the system **what** data you want
* You do **not** specify how to retrieve it

### Example

```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

### Why Declarative Matters

* The **database optimizer** chooses the best execution plan
* Query stays short and expressive
* Engine improvements automatically speed up existing queries
* Database is free to:

  * reorder operations
  * push down filters
  * reorganize storage
  * parallelize work
  * use indexes or not

---

### **Why Declarative is Powerful**

### **1. Hides Implementation Details**

You describe the desired result; the DB figures out execution.
→ Faster engines require **zero changes** to your queries.

### **2. Not Tied to Storage Order**

Regardless of how tables or indexes are stored:

* queries continue to work
* DBs freely reorganize data for performance

### **3. Better Parallelization**

Declarative queries avoid specifying steps, letting the engine:

* break work into parallel stages
* distribute across CPU cores
* even split across machines in distributed databases

---

### **Web Analogy: CSS (Declarative) vs JavaScript DOM Manipulation (Imperative)**

### **CSS (Declarative)**

```css
li.selected > p { background-color: blue; }
```

CSS expresses **what** should be styled.
The browser:

* finds matching elements
* applies/remove styles as state changes
* performs efficient reflows and repaints

### **JavaScript DOM (Imperative)**

You must manually:

* search DOM nodes
* loop through elements
* check conditions
* apply style changes

Declarative logic automatically adapts when state changes.
Imperative logic must be manually updated and maintained.

---

### **MapReduce in MongoDB**

MapReduce allows you to define custom computation logic:

* You provide small **map()** and **reduce()** functions
* Executes across large numbers of documents
* Very flexible but:

  * harder to optimize
  * slower
  * more code

MongoDB later introduced the **Aggregation Pipeline**, a more declarative style similar to SQL’s GROUP BY.

### **Example — MapReduce**

```js
db.observations.mapReduce(
  function map() {
    var year = this.observationTimestamp.getFullYear();
    var month = this.observationTimestamp.getMonth() + 1;
    emit(year + "-" + month, this.numAnimals);
  },
  function reduce(key, values) {
    return Array.sum(values);
  },
  {
    query: { family: "Sharks" },
    out: "monthlySharkReport"
  }
);
```

### **Example — Aggregation Pipeline (Declarative)**

```js
db.observations.aggregate([
  { $match: { family: "Sharks" } },
  { 
    $group: {
      _id: {
        year: { $year: "$observationTimestamp" },
        month: { $month: "$observationTimestamp" }
      },
      totalAnimals: { $sum: "$numAnimals" }
    }
  }
]);
```

---

## Graph-Like Data Models — Why and When They Matter

Most databases handle **one-to-many** relationships well.

But when your data involves:

* many-to-many
* dynamic connections
* complex multi-hop paths

…document databases and relational databases become awkward and inefficient.

Graph databases shine in these cases.

---

# ## **What Is a Graph Data Model?**

A graph contains:

| Component                 | Meaning                                                 |
| ------------------------- | ------------------------------------------------------- |
| **Vertices (Nodes)**      | Entities (Person, City, Product, Event)                 |
| **Edges (Relationships)** | Connections (friend_of, lives_in, purchased, linked_to) |

### Common examples:

* Social networks
* Road networks
* Recommender systems
* Knowledge graphs
* Biological networks

---

# ## **Key Strength of Graphs**

Graphs excel at **multi-hop traversals**, for example:

> “Find friends of friends who live in London and like surfing.”

In SQL → messy joins
In Graph DB → natural traversal

**Graph databases model relationships as first-class citizens.**

---

# ## **Property Graph Model (Neo4j, JanusGraph, Titan)**

Each **vertex** includes:

* unique ID
* properties (key-value pairs)
* incoming edges
* outgoing edges

Each **edge** includes:

* unique ID
* start vertex (tail)
* end vertex (head)
* relationship type
* properties (optional)

### Important qualities

1. **Flexible schema** – you can connect anything to anything
2. **Fast traversal** – edges are stored with nodes, enabling fast hops
3. **Multiple relationship types** – keeps models rich and clean

---

### **Real-World Example: Facebook**

Vertices:

* People
* Posts
* Events
* Locations
* Comments

Edges:

* friend_of
* liked
* authored
* attended
* checked_in

Allows complex queries like:

> “Show posts liked by friends of people who attended the same event as me.”

A natural fit for graph traversal.

---

### **Why Graphs Are Great for Evolving Data**

As your app grows and you add:

* allergies
* ingredients
* diets
* relationships
* preferences

You can simply:

* create new vertices
* connect new edges

No schema migration or denormalization needed.

---

### **When to Use a Graph Database**

Graphs are ideal when:

* Relationships are central
* Queries traverse multiple hops
* Structure evolves frequently
* You need expressive models for complex networks

If relationships are simple → relational/document DB is fine.
If relationships are rich → graph DB is more expressive **and** faster.

---

## Cypher: Declarative Querying for Graphs

Cypher is a **declarative** graph query language (Neo4j, Memgraph, etc.).
You describe *patterns*, not procedures.

### Example

```
MATCH (p:Person)-[:BORN_IN]->(place)-[:WITHIN*0..]->(country {name: "United States"})
RETURN p.name;
```

Highlights:

* `(:Label)` → node
* `[:RELATIONSHIP]` → edge
* `*0..` → traverse relationships of any depth
* Reads like ASCII-art

---

### **Why Graph Queries Are Hard in SQL**

SQL has no built-in graph traversal.
To express multi-hop relationships, you need **recursive CTEs**:

```sql
WITH RECURSIVE region_chain(place, country) AS (
  SELECT place, country FROM location WHERE country = 'United States'
  UNION ALL
  SELECT l.place, r.country
  FROM location l
  JOIN region_chain r ON l.parent = r.place
)
SELECT name
FROM person
JOIN region_chain ON person.birthplace = region_chain.place;
```

More verbose, less expressive, and harder to optimize.

---

## SPARQL and Triple Stores

Triple stores represent facts as:

```
subject — predicate — object
```

SPARQL offers graph-like querying with natural path traversal.

### Example

```
SELECT ?person
WHERE {
  ?person :bornIn / :within* :UnitedStates .
}
```

`/ :within*` means "follow this relationship zero or more times".

---

## Datalog and Rule-Based Graph Reasoning

Datalog expresses logic using **rules**, not procedural code.

### Example Rule

```
within_recursive(X, Y) :- within(X, Z), within_recursive(Z, Y).
```

This infers hierarchical relationships of **any depth**.
Datalog heavily influenced Cypher and SPARQL recursion.