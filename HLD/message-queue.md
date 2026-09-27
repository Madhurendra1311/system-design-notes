Messaging Queues are widely used in asynchronous systems. Message processing in an asynchronous fashion allows the client to relieve itself from waiting for a task to complete and, hence, can do other jobs during that time. It also allows a server to process its jobs in the order it wants to.

Messaging Queues provide useful features such as persistence, routing, and task management. 

A system having a message queue can move to higher-level requirements while abstracting implementation details of message delivery and event handling to the messaging queue.

The 'queue' is just a name for this data structure. In practice, it could be storing messages using any policy. Some examples of message queues are Kafka and RabbitMQ. They are widely used for various purposes, such as command query request segregation (CQRS) and event sourcing.

Notes to self:
* Servers are processing jobs in parallel.
* A server can crash. The jobs running on the crashed server still needs to get processed.
* A notifier constantly polls the status of each server and if a server crashes it takes ALL unfinished jobs (listed in some database) and distributes it to the rest of the servers. Because distribution uses a load balancer (with consistent hashing) duplicate processing will not occur as job_1 which might be processing on server_3 (alive) will land again on server_3, and so on.
* This "notifier with load balancing" is a "Message Queue".