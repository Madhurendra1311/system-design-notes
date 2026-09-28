Event Driven Systems pass and persist events. They have evolved from the publisher-subscriber model, and the design has some advantages. Events are immutable and can be replayed to allow the systems to take snapshots of their behavior. This allows services to 'self heal' as explained in the video.

A lot of transaction issues are alleviated once idempotency and retrial logic is added to a system. The system can retry messages an infinite number of times to the recipient till there is a message acceptance and acknowledgment from the receiver.

Event-driven systems are closely related to event sources and CQRS. Greg Young and Martin Fowler have been talking about these systems for a while. Events are persisted in something like a message queue, and hence the responsibility for retrial and persistence is moved to it. 

These abstractions enable the programmer to focus on the business logic of a system and add subscribers to events with minimum coupling with other services. Decoupling the system is one of the advantages of event-driven systems.

One major disadvantage of this system is that it is difficult to reason about the flow of a request. Services can independently register for an event and consume it without the publisher being aware of it.

We talk about different applications using an event-driven architecture such as Git and Gaming Systems. We then discuss the advantages and disadvantages of such an architecture (Event Sourcing).