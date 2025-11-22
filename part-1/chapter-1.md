# Reliable, Scalable, and Maintainable Applications

A data-intensive application is built from common building blocks:

* **Databases** → Store and retrieve data
* **Caches** → Speed up reads by keeping results of expensive operations
* **Search indexes** → Keyword search + filtering
* **Stream processors** → Asynchronous message handling
* **Batch processors** → Periodic crunching of large historical datasets

While it sounds simple, when building an application, we still need to figure out which tools and which approaches are the most appropriate for the task at hand. And it can be hard to combine tools when you need to do something that a single tool cannot do alone.

This book focuses on three core pillars:

* **Reliability** — The system should continue to work correctly (performing the correct function at the desired level of performance) even in the face of adversity (hardware or software faults, and even human error)
* **Scalability** — As the system grows (in data volume, traffic volume, or complexity), there should be reasonable ways of dealing with that growth.
* **Maintainability** — Over time, many different people will work on the system (engineering and operations, both maintaining current behavior and adapting the system to new use cases), and they should all be able to work on it productively

---

## 1. Reliability

A reliable system:

* The application performs the function that the user expected.
* It can tolerate the user making mistakes or using the software in unexpected ways
* Its performance is good enough for the required use case, under the expected load and data volume
* The system prevents any unauthorized access and abuse

### Faults vs Failures

* **Fault** → A component deviates from its spec
* **Failure** → The system stops providing the required service

Since faults can’t be eliminated, systems must be **fault-tolerant**. It is impossible to reduce the probability of a fault to zero; therefore it is usually best to design fault-tolerance mechanisms that prevent faults from causing failures. Counterintuitively, in such fault-tolerant systems, it can make sense to increase the rate of faults by triggering them deliberately—for example, by randomly killing individual processes without warning

### Chaos Engineering (Netflix)

Netflix intentionally introduces faults using **Chaos Monkey**, which:

* Randomly kills production servers
* Ensures the system recovers automatically
* Forces engineers to design for failure

 Chaos Monkey randomly shuts down production instances (servers or microservices) in Netflix’s cloud environment (AWS) during normal working hours. The goal is to make sure that:

* The system **can recover automatically**.
* The user experience **is not affected**.
* The engineering teams **design for failure**, not just for ideal conditions.

Part of the **Simian Army**:

| Tool                 | Purpose                         |
| -------------------- | ------------------------------- |
| Chaos Monkey         | Terminate instances randomly    |
| Latency Monkey       | Add artificial network delays   |
| Conformity Monkey    | Detect non-conforming instances |
| Doctor Monkey        | Monitor instance health         |
| Chaos Gorilla / Kong | Simulate AZ or region outages   |

---

### Types of Faults

#### 1. Hardware Faults

Hard disks fail, RAM corrupts, cables get unplugged.
Redundancy and multi-machine clusters reduce impact.

Cloud and scale increase failure frequency — so systems must tolerate node loss without downtime.

#### 2. Software Errors

Much more dangerous because they are **systematic** and **correlated** across nodes.

Examples:

* Kernel bug triggered by the 2012 leap second
* Runaway processes consuming CPU, memory, or disk
* Dependencies slowing down or returning corrupted data
* Cascading failures

The bugs that cause these kinds of software faults often lie dormant for a long time until they are triggered by an unusual set of circumstances. There is no quick solution to the problem of systematic faults in software.
Mitigation: good testing, isolating processes, measuring in production, crashing and restarting faulty processes.

#### 3. Human Errors

Humans make mistakes — design systems to minimize and recover from them.

Best practices:

* Safe abstractions and clear APIs
* Sandboxed environments
* Strong testing
* Easy rollbacks and gradual rollouts
* Monitoring and telemetry
* Training + good operational practices

Reliability matters for everything — not just nuclear plants.
Downtime → lost revenue, broken SLAs, reputational damage.

---

## 2. Scalability

Scalability = **how a system behaves when load increases**.

Scalability is the term we use to describe a system’s ability to cope with increased load. Note, however, that it is not a one-dimensional label that we can attach to a system: it is meaningless to say “X is scalable” or “Y doesn’t scale.” Rather, discussing scalability means considering questions like “If the system grows in a particular way, what are our options for coping with the growth?” and “How can we add computing resources to handle the additional load?”

### Describing Load

Use **load parameters**:

* Requests/second
* Read/write ratio
* Active users in chat room
* Cache hit rate
* Message throughput

#### Twitter Example

1. **Post Tweet** → ~12k writes/sec
2. **Home Timeline** → ~300k reads/sec

Main challenge: **fan-out** (distributing a tweet to followers).

**Initial approach: Read-time fan-out**

* Store all tweets in one table. (Simple writes since tweet has to be written to one place)
* When a user views their timeline, fetch tweets from everyone they follow. (Expensive reads because of query on huge dataset and expensive joins)

**Improved approach: Write-time fan-out**

* When someone tweets, push that tweet into each follower’s home timeline cache (expensive writes if celebrity)
* Reads get very fast and inexpensive

**Final hybrid**

* Normal users → push tweets (Write-time fan-out)
* Celebrities → fetch on demand (Read-time fan-out)
Both these type of fetches are combined based on the account and provided

---

### Describing Performance

Two questions:

1. How does performance change as load increases?
2. How much hardware is needed to maintain performance as load increases?

Metrics:

* **Throughput** (batch): records/sec
* **Response time** (online): the time from request to response

#### Latency vs Response Time

* **Service time**: actual processing
* **Latency**: waiting in queue
* **Response time** = service time + latency + delays

#### Percentiles Matter

* **p50**: median
* **p95 / p99**: tail latencies
* Used in **SLIs/SLOs/SLAs**

Amazon example:

* +100 ms → -1% sales
* +1 second → -16% satisfaction

So systems often target p99 or p99.9 response times in their Service Level Objectives (SLOs) or Agreements (SLAs)

---

### Handling Load

Two strategies:

#### 1. Vertical Scaling (Scale Up)

* Bigger machine
* Simple, but limited and expensive

#### 2. Horizontal Scaling (Scale Out)

* Many small machines
* Shared-nothing architecture
* Harder for stateful systems

In practice, most systems use a hybrid approach — several moderately powerful machines, not one giant or many tiny ones:
**Elastic scaling** → auto-scale
**Manual scaling** → stable, predictable

#### Stateless vs Stateful

* Stateless → easy to scale (just add nodes) (e.g., APIs)
* Stateful (DBs) → harder; distributed systems help (e.g., databases)

No universal scalable architecture — depends on:

* Access patterns
* Read/write mix
* Data size
* Latency requirements

A system handling 100k small requests/sec is totally different from one handling 3 massive requests/min, even if both move the same total data volume. General-purpose patterns emerge: caches, queues, partitions, etc.

---

## 3. Maintainability

It is well known that the majority of the cost of software is not in its initial development, but in its ongoing maintenance—fixing bugs, keeping its systems operational, investigating failures, adapting it to new platforms, modifying it for new use cases, repaying technical debt, and adding new features.

Yet, unfortunately, many people working on software systems dislike maintenance of so-called legacy systems—perhaps it involves fixing other people’s mistakes, or working with platforms that are now outdated, or systems that were forced to do things they were never intended for. Every legacy system is unpleasant in its own way, and so it is difficult to give general recommendations for dealing with them. However, we can and should design software in such a way that it will hopefully minimize pain during maintenance, and thus avoid creating legacy software ourselves.

To avoid creating painful legacy systems, we focus on:

---

### 3.1 Operability

Make it easy for operations teams to run the system. It has been suggested that “good operations can often work around the limitations of bad (or incomplete) software, but good software cannot run reliably with bad operations”

Operations teams handle:

* Monitoring and alerting
* Restoring service during failures
* Debugging performance problems
* Patching and updates
* Deployment + configuration management
* Security
* Capacity planning
* Maintaining organizational knowledge

Good operability includes:

* Deep visibility and monitoring
* Automation and integration with standard tools
* No single-machine dependency
* Good docs and predictable behavior
* Safe defaults + admin control
* Self-healing when possible

---

### 3.2 Simplicity

Reduce **accidental complexity** — Make it easy for new engineers to understand the system, by removing as much complexity as possible from the system.

Symptoms of complexity (“big ball of mud”):

* Special cases everywhere
* Hard-to-follow dependencies
* Workarounds that pile up

Aim for simplicity through **abstraction**:

* It hides low-level details behind a clean interface.
* Enables **reuse** and **improves quality** across systems.

Good abstractions simplify development but are hard to design, especially in distributed systems.

---

### 3.3 Evolvability

Systems must adapt to changing requirements.

Evolvable systems:

* Use clean modular designs
* Have good abstractions
* Are easy to modify and refactor
* Support Agile practices (TDD, CI/CD, refactoring)

Also known as: **extensibility, modifiability, or plasticity**.
