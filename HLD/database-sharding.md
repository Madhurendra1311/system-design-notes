Sharding a database is a common scalability strategy for designing server-side systems. The server-side system architecture uses concepts like sharding to make systems more scalable, reliable, and performant.

Sharding is the horizontal partitioning of data according to a shard key. This shard key determines which database the entry to be persisted is sent to. Some common strategies for this are reverse proxies.

Database interviews ask for concepts like sharding to make databases more performant and available. This makes horizontal partitioning a logical choice.

Notes to self:
* Sharding is basically a hiearchical way to index databases.
* One problem is that you have to split the database somehow. What do you split on?
* You only shard shards when the shard grow too big.
* When shard fails you use the master/slave architecture. Writes always go to master, reads are distributed across the slaves. When the master fails one of the slaves become master.