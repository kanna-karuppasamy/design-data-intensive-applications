# Encoding and Evolution

This chapter investigates the formats used for **encoding data** and the challenges that arise when systems and data formats **evolve** over time. It details how data flow impacts the compatibility and reliability of systems.

## The Challenge of Data Evolution

Applications inevitably change, requiring modifications to the stored data's format or schema. This change is complicated by the fact that the application code itself cannot be updated instantaneously across all parts of a distributed system:

* **Server-side Applications:** Use **rolling upgrades** (staged rollouts) where new code is deployed to a few nodes at a time to ensure service uptime and minimize risk.
* **Client-side Applications:** Depend on users installing updates, meaning old client versions can coexist with new server versions for long periods.

This simultaneous existence of old and new code, and old and new data, necessitates two key forms of compatibility:

1.  **Backward Compatibility:** Newer code must be able to read data that was written by older code. (Generally easier to achieve.)
2.  **Forward Compatibility:** Older code must be able to read (or safely skip) data that was written by newer code. (Often trickier, as older code must anticipate future, unknown additions.)

### Formats for Encoding Data

When processing data, a program uses two distinct representations, requiring translation between them:

1.  **In Memory:** Data is held in structures like objects, structs, arrays, and hash tables, optimized for **CPU access** (using pointers).
2.  **Over the Wire/On Disk:** Data must be encoded as a **self-contained sequence of bytes** (a file or network message), as pointers are meaningless outside the originating process.

The translation process is:

* **Encoding (Serialization/Marshalling):** Translating the in-memory representation to a byte sequence.
* **Decoding (Parsing/Deserialization/Unmarshalling):** Translating the byte sequence back to the in-memory representation.

---

Here is the content for the section on language-specific encoding formats.

---

### Language-Specific Formats

Many programming languages offer built-in or third-party libraries for encoding in-memory objects into a byte sequence (e.g., **Java's `java.io.Serializable`**, **Python's `pickle`**, **Ruby's `Marshal`**). While these are convenient for saving and restoring objects with minimal code, they are generally a poor choice for persistent storage or inter-service communication due to several deep problems:

* **Language Dependency:** The encoding is often tightly tied to a specific programming language. Reading the data in a different language is extremely difficult, creating a long-term **vendor lock-in** to the current programming language and making system integration with other organizations (which may use different languages) nearly impossible.
* **Security Problems:** The decoding process often requires the ability to instantiate arbitrary classes to restore the original object types. If an attacker can manipulate the input byte sequence, they can trick the application into instantiating malicious classes, which is a frequent source of security vulnerabilities (e.g., **Remote Code Execution**).
* **Versioning Issues:** Compatibility (forward and backward) is often an afterthought in these quick-and-easy libraries, making system evolution difficult.
* **Inefficiency:** Performance, in terms of both CPU time for encoding/decoding and the size of the resulting encoded data, is often poor. (For example, Java's built-in serialization is known for its slow performance and large, "bloated" encoding.)

For these reasons, it is a bad practice to use language-specific encoding formats for anything other than very **transient** purposes within a single application instance.

---

Here is the content for the section on JSON, XML, and Binary Variants.

-----

### JSON, XML, and Binary Variants

JSON, XML, and CSV are popular, standardized, language-independent textual formats.

  * **XML** is often criticized for being too verbose and complex.
  * **JSON** gained popularity due to its simplicity relative to XML and its built-in support in web browsers (as a subset of JavaScript).
  * **CSV** is simple but less powerful and lacks formal schema support.

#### Problems with Textual Formats

These formats, despite their widespread use, suffer from subtle problems:

1.  **Numeric Encoding Ambiguity:**
      * XML and CSV cannot distinguish between a string and a number without an external schema.
      * JSON distinguishes strings and numbers but does not distinguish integers and floating-point numbers, nor does it specify precision.
      * This is a problem for large integers (e.g., greater than $2^{53}$), which cannot be exactly represented by standard 64-bit IEEE 754 floating-point numbers (used by languages like JavaScript), leading to potential data loss.
      * *Workaround:* Companies like Twitter often include large IDs twice in their JSON response: once as a JSON number (for convenience) and once as a decimal string (for exact parsing).
2.  **Lack of Binary String Support:** They lack native support for sequences of bytes (binary data).
      * *Workaround:* Binary data is often encoded as text using **Base64**. This is "hacky" and increases the data size by 33%.
3.  **Optional/Complex Schemas:**
      * While XML and JSON have complex, powerful schema languages (XML Schema, JSON Schema), many tools skip using them.
      * Without a schema, the correct interpretation of data (e.g., distinguishing a number from a string, or identifying Base64-encoded binary data) often has to be **hardcoded** into the application logic.
4.  **CSV Ambiguity:** CSV is a vague format whose interpretation depends on the application. Its lack of a schema and inconsistent escaping rule implementation by parsers can lead to errors.

Despite these issues, these formats are effective for **data interchange** between organizations, where widespread agreement on the format outweighs concerns about efficiency or elegance.

-----

#### Binary Encoding

For data used internally within an organization, formats that are more **compact** and **faster to parse** are often preferred.

  * Both JSON and XML are verbose, leading to a profusion of specialized **binary encodings** (e.g., MessagePack, BSON for JSON; WBXML, Fast Infoset for XML).
  * These binary formats generally maintain the JSON/XML data model but include better datatype support (e.g., specific integer types, binary strings).
  * Crucially, because these formats typically do not enforce a schema, they still must include **all object field names** within the encoded data.

**Example: MessagePack**
When encoding a record like:

```json
{
 "userName": "Martin",
 "favoriteNumber": 1337,
 "interests": ["daydreaming", "hacking"]
}
```

MessagePack includes type indicators and lengths before the data itself. For example, the first byte `0x83` indicates an object with three fields, and subsequent bytes include the full field names (e.g., `userName`).

  * The resulting binary encoding (66 bytes in the example) is only marginally shorter than the textual JSON (81 bytes without whitespace). The minimal space reduction and parsing speedup are often questioned as being worth the loss of human-readability.

The next generation of binary formats will show how to achieve much greater compression by leveraging schemas.

-----

### Thrift and Protocol Buffers

**Apache Thrift** (developed at Facebook) and **Protocol Buffers (protobuf)** (developed at Google) are highly efficient, schema-driven binary encoding frameworks built on the same core principle.

#### The Schema and Code Generation

Both frameworks require a schema defined using an Interface Definition Language (IDL):

  * **Thrift Example (IDL):**
    ```thrift
    struct Person {
      1: required string userName,
      2: optional i64 favoriteNumber,
      3: optional list interests
    }
    ```
  * **Protocol Buffers Example:**
    ```protobuf
    message Person {
      required string user_name = 1;
      optional int64 favorite_number = 2;
      repeated string interests = 3;
    }
    ```

A code generation tool takes this definition and produces classes in various languages (Java, C++, Python, etc.) that implement the schema, including efficient methods for encoding and decoding records.

#### Binary Encoding: Tag Numbers

The key to the efficiency of Thrift and Protobuf compared to JSON/XML binary formats is the absence of verbose field names in the encoded data.

1.  **Tag Numbers:** The schema assigns a unique, non-changeable **tag number** (e.g., `1`, `2`, `3`) to each field.
2.  **Compact Representation:** The encoded record contains only the **tag number** and a type annotation, followed by the field value. This significantly reduces message size.

For instance, the example record (81 bytes in JSON, 66 bytes in MessagePack) can be encoded in approximately **33–34 bytes** using their compact binary formats.

  * **Thrift CompactProtocol** (34 bytes) achieves compression by packing the field type and tag number into a single byte, and by using **variable-length integers** (varint).
  * **Protocol Buffers** (33 bytes) is very similar, using techniques like **varint** to encode small numbers efficiently (e.g., small integers are encoded in just 1 or 2 bytes, rather than a full 8 bytes).

#### Schema Evolution with Tag Numbers

The tag number is critical for schema evolution, as its meaning (what field it represents) is constant.

  * You can change a field's **name** in the schema, but you **cannot change its tag number**.
  * **Adding Fields:** New fields must be assigned a new tag number.
      * **Forward Compatibility:** Old code (which doesn't know the new tag) simply **skips** the unknown field based on its type annotation and continues parsing.
      * **Backward Compatibility:** New fields **cannot be marked as required** (they must be `optional` or have a default value), because old data written without the new field would fail the required check when read by new code.
  * **Removing Fields:** You can only remove an `optional` field. The tag number can never be reused, as old data might still exist with that tag, and new code must be able to skip it.
  * **Changing Datatypes:** Generally possible (e.g., 32-bit integer to 64-bit integer) as long as the encoding is compatible. However, downgrading (e.g., 64-bit to 32-bit) risks truncation if old code reads a new, large value.

**Protocol Buffers Detail (Repeated Fields)**
Protobuf does not have a dedicated list type; instead, it uses a `repeated` marker. A `repeated` field is encoded by simply including the same field tag multiple times. This allows for safe schema evolution where an `optional` (single-valued) field can be changed into a `repeated` (multi-valued) field. Old code reading the new data will simply see the *last* value in the list.

-----

### Avro

**Apache Avro** is a binary encoding format that differs significantly from Protocol Buffers and Thrift, specifically designed to meet the use cases of Hadoop and large data processing.

#### Schema and Encoding

  * **Schema Definition:** Avro uses two schema languages: a human-editable Avro IDL and a machine-readable JSON representation. Critically, Avro schemas **do not use tag numbers**.
      * *JSON Example:*
        ```json
        {
         "type": "record",
         "name": "Person",
         "fields": [
          {"name": "userName", "type": "string"},
          {"name": "favoriteNumber", "type": ["null", "long"], "default": null}
         ]
        }
        ```
  * **Encoding:** Without tag numbers, the encoded data is simply the **concatenation of values**.
      * The encoding is the most compact of all seen formats (32 bytes for the example record).
      * **Crucial Implication:** The binary data can only be decoded correctly if the reader knows the exact schema used by the writer, as the data itself contains no field identifiers or datatype markers.

#### Schema Evolution via Resolution

Avro supports schema evolution by leveraging a key concept: the **writer's schema** and the **reader's schema**.

  * **Writer's Schema:** The schema used by the application that encodes and writes the data.
  * **Reader's Schema:** The schema expected by the application that decodes and reads the data (compiled into the application).
  * **Resolution:** When data is decoded, the Avro library takes both the writer's schema and the reader's schema, resolves their differences, and **translates** the data from the writer's schema to the reader's schema.
      * Fields are matched by **name**, not by tag number.
      * Fields present in the writer's schema but not the reader's are **ignored**.
      * Fields expected by the reader but missing from the writer's schema are filled in with the **default value** declared in the reader's schema.

#### Compatibility Rules

To maintain both **backward** (New Reader $\to$ Old Data) and **forward** (Old Reader $\to$ New Data) compatibility:

| Action | Compatibility Rule |
| :--- | :--- |
| **Add/Remove Field** | The field **must** have a default value. |
| **Change Data Type** | Possible, provided Avro can perform an automatic conversion (e.g., small integer $\to$ large integer). |
| **Change Field Name** | Possible, but the reader's schema must explicitly define an **alias** for the old name. |

> **Union Types:** Avro uses `union` types (e.g., `union { null, long }`) to explicitly specify if a field can be null. A default value can only be used if it is a branch of the union.

#### How the Reader Knows the Writer's Schema

Since the schema isn't included with every field, the reader must know which schema version was used by the writer. The method depends on the context:

1.  **Large File (e.g., Hadoop):** The writer includes the schema once at the **beginning of the file** (using an Avro **object container file**). The file is then self-describing.
2.  **Database Records:** Each record can include a small **schema version number**. The reader uses this number to look up the full writer's schema from a central **schema database**.
3.  **Network Connection (RPC):** The two processes **negotiate** the schema version upon connection setup.

#### Dynamically Generated Schemas

Avro's reliance on **field names** rather than manually assigned **tag numbers** makes it ideal for dynamically generated schemas.

  * **Use Case:** Automatically generating an Avro schema from a changing relational database schema (where column names become field names).
  * The data export process can generate a new schema every time it runs. Readers only need the name-based resolution mechanism to match the new writer's schema against their old reader's schema.
  * This removes the administrative burden of manually maintaining and assigning field tags, which would be necessary with Protocol Buffers or Thrift.

#### Code Generation

  * Avro provides **optional code generation** for statically typed languages (Java, C++).
  * Crucially, it can also be used **without code generation** in dynamically typed languages (Python, Ruby, data processing languages like Apache Pig). Since the object container file is **self-describing** (it includes the writer's schema), the data can be read and analyzed directly without a separate compilation step.

-----

### The Merits of Schemas

Formats like Protocol Buffers, Thrift, and Avro rely on a schema to define a compact binary encoding. While their schema languages are simpler than complex validation standards like XML Schema or JSON Schema, they have become widespread due to their simplicity, performance, and strong support across many programming languages.

These modern schema-based binary formats share principles with older standards like **ASN.1** (used for SSL certificates), but they offer several significant advantages over textual formats (JSON, XML, CSV) or binary formats that omit schemas (BSON, MessagePack):

* **Compactness:** They are much more **compact** than "binary JSON" variants because they use field **tag numbers** or **positional field names** (Avro) instead of including the verbose field names in the encoded data.
* **Documentation and Guarantee:** The schema serves as **up-to-date documentation** for the data structure. Because the schema is **required for decoding**, you can be certain that the documentation accurately reflects the data being read.
* **Compatibility Checking:** By maintaining a database of schemas, you can automatically **check forward and backward compatibility** of a proposed schema change *before* deploying it, preventing system breaks.
* **Statically Typed Languages:** For languages like Java or C++, the ability to **generate code** from the schema enables crucial compile-time type checking, improving reliability and developer tooling.

In essence, schema-based binary encoding provides the flexibility of **schema-on-read** (allowing old and new data formats to coexist) while simultaneously offering superior guarantees and tooling associated with a formally defined structure.

---

## Modes of Dataflow

When data is sent between processes that do not share memory (e.g., across a network or to a file), it must be **encoded** as a sequence of bytes. The goal of encoding compatibility (forward and backward) is to allow different parts of the system to be upgraded independently, ensuring **evolvability**.

The next sections examine three common ways data flows between processes:

* Via databases.
* Via service calls (REST and RPC).
* Via asynchronous message passing.

---

### Dataflow Through Databases

In this mode, the process that **writes** to the database encodes the data, and the process that **reads** from the database decodes it.

#### Compatibility Requirements

1.  **Backward Compatibility (New Code Reads Old Data):** This is essential. A newer version of the application code must be able to read data written by its older self.
2.  **Forward Compatibility (Old Code Reads New Data):** This is also frequently required, especially during a **rolling upgrade** where older application instances may still be running and need to read data recently written by a newer version.

#### The Update/Rewrite Problem

A subtle challenge arises when an older version of the application reads a record, updates it, and writes it back to the database:

* If the new data contained a field unknown to the old code (forward compatibility required), the old code may **silently drop (lose) the unknown field** when it re-encodes and writes the record back.
* **Solution:** The application must be designed to **preserve unknown fields** through the decode-update-re-encode cycle. 

#### Data Outlives Code

The most significant consideration for database dataflow is that **data outlives code**.

* Server code can be replaced entirely in minutes, but the data stored in the database can remain in its original encoding for **years**.
* **Data Migration (Rewriting):** Explicitly rewriting all existing data into a new schema is possible but **expensive** on large datasets. Most databases (especially relational ones) avoid full migration for simple schema changes (e.g., adding a nullable column) by having the database management system (DBMS) fill in `null` values for old rows on the fly.
* Schema evolution allows the entire database to appear as if it was encoded with a single, current schema, even though the underlying storage may contain records encoded with various historical versions.

#### Archival Storage

When data is copied out of the database for backups or into a data warehouse, it is typically rewritten using the **latest schema** in a consistent format (e.g., Avro Object Container Files) and often encoded into an **analytics-friendly column-oriented format** (e.g., Parquet).

---

### Dataflow Through Services: REST and RPC

Dataflow through services involves **clients** making requests to **servers** that expose an application-specific API. This approach is fundamental to **Service-Oriented Architectures (SOA)** and **Microservices**.

#### Web Services: REST vs. SOAP

When HTTP is used as the transport protocol, the service is often called a **web service**. Two popular, contrasting approaches dominate:

| Feature | REST (Representational State Transfer) | SOAP (Simple Object Access Protocol) |
| :--- | :--- | :--- |
| **Philosophy** | Design philosophy built on HTTP principles (URLs, caching, content negotiation). | XML-based protocol, often used over HTTP but attempts to be HTTP-independent. |
| **Data Format** | Typically JSON, though any simple format is used. | Always XML (with WSDL for description). |
| **Complexity** | Simpler, favors less tooling and code generation. | Complex, relies heavily on tools, code generation, and a sprawling set of WS-\* standards. |
| **Popularity** | Dominant for public and cross-organizational APIs; favored in microservices. | Still used in large enterprises but has fallen out of favor elsewhere. |

#### The Flaws of Remote Procedure Calls (RPC)

RPC attempts to make a network request look like a local function call (**location transparency**), but this abstraction is fundamentally flawed because network calls are vastly different from local calls:

| Issue | Local Function Call | Network RPC |
| :--- | :--- | :--- |
| **Reliability** | Predictable: succeeds or fails. | Unpredictable: request/response may be lost, machine may be slow/unavailable. |
| **Failure Mode** | Returns result or throws exception/crashes. | **Timeout:** You may not know if the request succeeded or failed on the remote server. |
| **Retry Safety** | Not needed. | Retries can cause the action to be performed multiple times unless **idempotence** (deduplication) is built in. |
| **Latency** | Consistent and fast. | Wildly variable and much slower. |
| **Data Passing** | Efficiently pass object **references** (pointers). | Requires encoding/decoding all parameters into a byte sequence. |
| **Language** | Single language/datatype system. | Requires translating datatypes between potentially different languages (e.g., handling JavaScript's $2^{53}$ number limit). |

#### Current Directions in RPC

Despite its flaws, RPC is still widely used internally within organizations (e.g., within a datacenter) for performance and ease of use. Modern RPC frameworks (like gRPC, Finagle, Avro RPC) are more explicit about network failures by:

* Using **futures** (promises) to encapsulate asynchronous, fallible actions.
* Supporting **streams** (series of requests/responses over time).
* Providing **service discovery** to locate services.

**Key Distinction:** **RESTful APIs** remain the predominant choice for **public APIs** due to their human-readability, ease of debugging, and massive ecosystem of tools. **Custom binary RPC protocols** (using Thrift, Protobuf, Avro) are mainly focused on high-performance communication **between internal services**.

#### Compatibility and Evolution for Services

For service evolvability, a key assumption can often be made: all servers are typically updated first, followed by all clients.

* This means **Backward Compatibility** is required for **requests** (new server reads old client request).
* **Forward Compatibility** is required for **responses** (old client reads new server response).

Compatibility rules are inherited from the encoding format used:

* **Schema-Based (Thrift, gRPC, Avro RPC):** Compatibility is handled reliably by the schema's rules (tag numbers or field names) for adding/removing fields.
* **REST/JSON:** New optional request parameters and new response fields are generally considered compatible changes.
* **API Versioning:** Because service providers often cannot force clients to upgrade, compatibility must be maintained for a long time. Compatibility-breaking changes usually require maintaining **multiple versions** of the API side by side (e.g., using a version number in the URL or the HTTP `Accept` header).

---

### Message-Passing Dataflow

Asynchronous message-passing systems sit between synchronous RPC and persistent databases. They use an intermediary called a **message broker** (or message queue) that temporarily stores a message before delivering it to one or more recipients.

| Feature | Message-Passing System | Direct RPC/REST |
| :--- | :--- | :--- |
| **Delivery** | Asynchronous (sender doesn't wait for reply). | Synchronous (sender waits for immediate reply). |
| **Path** | Via an intermediary (message broker/queue). | Direct network connection. |
| **Advantage** | Decoupling, buffering, automatic redelivery, fan-out (one message to many recipients). | Low latency, immediate feedback (response). |

#### Message Brokers

Modern message brokers include open-source systems like **RabbitMQ**, **ActiveMQ**, and **Apache Kafka**.

* **Operation:** A producer sends a message to a named **queue** or **topic**. The broker delivers the message to one or more consumers/subscribers.
* **Data Model:** Messages are typically just a sequence of bytes with metadata, allowing the use of **any encoding format**.
* **Compatibility:** Using a backward and forward compatible encoding provides the greatest flexibility to change producers and consumers independently.
* **Preserving Unknown Fields:** If a consumer processes a message and then republishes it to another topic, care must be taken (as in databases) to **preserve unknown fields** to avoid data loss.

#### Distributed Actor Frameworks

The **actor model** is a programming pattern for concurrency where logic is encapsulated in isolated, communicating **actors**.

* **Distributed Extension:** Distributed actor frameworks (like Akka and Erlang OTP) extend this model across multiple nodes, using the same message-passing mechanism regardless of location.
* **Location Transparency:** This works better than in RPC because the actor model inherently assumes message loss is possible, even locally.
* **Encoding Challenges:** Distributed actors still face compatibility challenges during rolling upgrades, as messages may flow between new and old versions of the code.

| Framework | Default Encoding | Compatibility Status |
| :--- | :--- | :--- |
| **Akka** | Java's built-in serialization (poor compatibility). | Can be replaced with Protocol Buffers or another compatible format. |
| **Orleans** | Custom format (doesn't support rolling upgrades by default). | Custom serialization plug-ins can be used. |
| **Erlang OTP** | Changes to record schemas are hard, requiring careful planning for rolling upgrades. |

---

## Summary

This chapter explored the methods and implications of turning in-memory data structures into a sequence of bytes for network transfer or disk storage (**encoding**).

The need for **backward compatibility** (new code reads old data) and **forward compatibility** (old code reads new data) is paramount, particularly to support **rolling upgrades**—the ability to deploy a new service version gradually without downtime or coordination.

### Data Encoding Formats

| Format Type | Examples | Key Properties |
| :--- | :--- | :--- |
| **Language-Specific** | Java Serialization, Python pickle | Restrictive (single language), poor security, poor compatibility, often inefficient. **Avoid for persistent use.** |
| **Textual** | JSON, XML, CSV | Widespread, human-readable, good for interchange. **Vague datatypes** (numbers, binary strings) and **optional/complex schemas** can be problematic. |
| **Binary Schema-Driven** | Thrift, Protocol Buffers, Avro | **Compact** (omits field names), highly **efficient**, and provides **clear compatibility semantics** via schemas (tag numbers or field names). Supports code generation. |

### Modes of Dataflow

| Dataflow Mode | Sender/Encoder | Recipient/Decoder | Key Compatibility Concern |
| :--- | :--- | :--- | :--- |
| **Databases** | Writer (Current Version) | Reader (Future Version) | Data **outlives code**, requiring long-term backward and forward compatibility, and care to **preserve unknown fields** during read-update-write cycles. |
| **Services (REST/RPC)** | Client (Request), Server (Response) | Server (Request), Client (Response) | Requires backward compatibility for requests and forward compatibility for responses; difficult to enforce upgrades, often necessitates multi-version API support. |
| **Message Passing** | Producer | Consumer | Flexibility to upgrade independently requires highly compatible encoding (e.g., Avro), and preservation of unknown fields if messages are republished. |

By using schema-driven binary encodings and carefully following compatibility rules, achieving evolvability and frequent, low-risk deployments is highly achievable.

---