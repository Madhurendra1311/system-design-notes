NoSQL is a popular database storage method. It keeps data as key-value pairs. The advantages and disadvantages of NoSQL compared with RDBMS (which uses SQL) are discussed here, using the Cassandra architecture as an example.

NoSQL vs RDBMS (SQL) — Advantages and Disadvantages
NoSQL is a type of database designed to store and manage large amounts of structured, semi-structured, or unstructured data. Unlike relational databases (RDBMS), which store data in tables with rows and columns, NoSQL databases support different data models, such as key-value, document, column-family, and graph.
Apache Cassandra is an example of a wide-column NoSQL database designed for high availability, scalability, and distributed data storage.
Advantages of NoSQL over RDBMS
1. High scalability: NoSQL databases can scale horizontally by adding more servers to handle increasing data and traffic.
2. High availability: Cassandra replicates data across multiple nodes, helping the system remain available even if some nodes fail.
3. Flexible schema: NoSQL databases can handle changing data structures without requiring a fixed table schema.
4. High performance: NoSQL can provide fast read and write operations for workloads designed around its data model.
5. Distributed architecture: Data can be distributed across multiple data centers and geographical locations.
6. Large-scale data handling: Suitable for applications that process huge volumes of data, such as IoT platforms, real-time analytics, and activity tracking.
Disadvantages of NoSQL compared with RDBMS
1. Limited joins: Many NoSQL databases do not support relational joins in the same way as traditional SQL databases.
2. Data duplication: Data may need to be stored in multiple places to optimize queries.
3. Complex transactions: Transaction support varies by NoSQL database; some complex multi-record transactions are easier to implement in RDBMS.
4. Consistency trade-offs: Distributed NoSQL systems may offer different consistency options. Cassandra, for example, supports tunable consistency.
5. Query limitations: Query capabilities and standards vary between NoSQL databases, whereas SQL is widely standardized.
6. Data modeling complexity: Developers must design data models around application access patterns, which can make future query changes more difficult.
Cassandra vs RDBMS comparison
Feature	Cassandra (NoSQL)	RDBMS (SQL)
Data model	Wide-column	Tables, rows, columns
Schema	Defined table schema, with flexible evolution	Typically predefined relational schema
Scaling	Horizontal scaling	Traditionally vertical; horizontal options also exist
Relationships	Usually modeled without joins	Supports relationships and joins
Availability	Distributed, peer-to-peer architecture	Depends on configuration and replication
Transactions	Supports lightweight transactions and atomic operations within defined scopes	Often supports rich ACID transactions
Best suited for	High-volume distributed workloads	Relational data and complex queries
How Cassandra architecture works
- Nodes: Individual servers that store data.
- Cluster: A group of nodes working together.
- Replication: Copies data across multiple nodes to improve fault tolerance.
- Partitioning: Distributes data among nodes using a partition key.
- Peer-to-peer architecture: Nodes communicate without relying on a single master node.
Conclusion: NoSQL databases such as Cassandra are excellent for highly scalable, distributed applications that require high availability and large write throughput. RDBMS databases are often a better choice when applications require complex joins, strong relational integrity, and sophisticated transactions. The right choice depends on the application's data model, query patterns, consistency requirements, and scale.

We talk about sharding, redundancy, load balancing, compaction and other features in NoSQL databases. This allows them to scale efficiently.

NoSQL explanation and comparison
Cassandra Architecture
Quorum
Compaction of SST tables

Note:
Not all NoSQL store data as a key-value pair. What you are talking about is only a subset of NoSQL. There are 4 meain types of NoSQL databases: Document, key-value, wide-column and graph.

Two corrections
1) Cassandra do not store log file in-memory. Rather it stores on disk. That's how it can recover from during failures
2) Cassandra do not append in-memory it rather appends to commit log on disk. So, in-memory the data is sorted Memtable. When it reaches certain limit then it flushes to the disk on SSTable


A small point to add, Most of the NoSQL offerings also offer, Consistency levels for users to choose. So, If I want to make sure my users read always consistent data I can have strong consistency which means a write is acknowledged only when a quorum of replicas have  also acknowledged it. This makes sure,  consistency is present even when one of the replicas go down. But obviously the tradeoff is the writes are slow. If availability is preferred over consistency,  then Eventual consistency can be choosen in which the writes are acknowledged when the present replica writes it in memory,  hoping that all other replicas catch up with the write "eventually" .


