Data Consistency and Tradeoffs in Distributed Systems (HLD Notes)

1. What is consistency?
Consistency means that every reader sees the same, correct data, no matter which node answers them. In a distributed system the hard part is keeping that promise while data is split up, copied, and spread across machines that can fail.
The simplest case is one node holding one copy of the data, such as an account balance in a single database. Every read returns the latest write, because there is nothing else to disagree with.
• Reads: always return the most recent committed write.
• Writes: applied one at a time, in a clear order.
• Transactions: either fully happen or not at all (the atomicity part of ACID).
Everything that follows in these notes is about what breaks when we leave this simple setup, and what we pay to fix it.

2. Single node problems
A single node is easy to keep consistent, but it cannot survive real traffic or real failures. It has three limits.
• Single point of failure: if the machine crashes or its disk dies, the data is unavailable or lost.
• Throughput limit: one machine has finite CPU, memory, and disk I/O. Past a certain request rate, latency climbs and requests time out.
• Storage limit: the data may simply not fit on one machine.
Scaling up (a bigger machine) only delays the problem. The real fix is to scale out across several nodes, and that is where consistency gets hard.

3. Splitting the data and the problems with disjoint data
The first way to scale out is to split the data into disjoint parts, called shards or partitions. Each node owns a different slice, for example users A–M on one node and N–Z on another. Reads and writes for a given key go to exactly one node, so there is still only one copy of each record.
This fixes throughput and storage, but creates new problems:
• Multi-shard operations: a transfer from account X (node 1) to account Y (node 2) touches two nodes. If node 1 debits and node 2 crashes before crediting, money disappears.
• Atomicity across nodes: a local transaction cannot guarantee that both writes happen or neither does.
• Uneven load: a popular key or a poorly chosen shard key creates a hot node.
• Still no fault tolerance: if one node dies, its slice of the data is gone.
The last point leads directly to keeping copies of the data.

4. Data copies (replication)
To survive node failures and spread read load, we keep several copies (replicas) of the same data on different nodes. Now the same record lives in more than one place, so the copies can disagree.
• Replica lag: a write lands on replica A, but replica B has not received it yet. A client reading from B gets stale data.
• Conflicting writes: two clients write different values to the same key on different replicas at the same time. Which one wins?
• Partial failure: a write succeeds on some replicas and fails on others.
There are two broad ways to propagate a write:
Approach
How it works
Consistency
Cost
Synchronous
Wait for all (or a quorum of) replicas to confirm before acknowledging
Strong: replicas agree
Higher write latency; blocked if a replica is down
Asynchronous
Acknowledge after the first node writes, copy to others in the background
Weak: replicas may lag
Low latency, but data can be lost or stale
Synchronous replication needs the nodes to agree, which is harder than it sounds.

5. The two generals problem
Two nodes cannot be perfectly certain they agree if the network between them can drop messages. This is the two generals problem, and it has no perfect solution.
Two generals camp on opposite sides of a valley and must attack together. Messengers cross the valley and can be captured. General A sends "attack at dawn." A does not know whether it arrived, so A waits for an acknowledgment. B sends one, but B does not know whether the acknowledgment arrived, so B needs an acknowledgment of that too. The chain never ends.
The lesson for distributed systems:
• A node that sends a message and hears nothing cannot tell whether the message was lost, the receiver crashed, or only the reply was lost.
• Guaranteeing that all nodes agree on every operation over an unreliable network is impossible in the strict sense.
• In practice we accept a very high probability of agreement using retries, timeouts, and acknowledgments, and design for the rare case where it fails.

6. Leader assignment
One way to avoid conflicting writes is to name one node the leader for each piece of data. All writes go to the leader, which orders them and forwards them to followers. Followers serve reads or stand by as backups.
This removes write conflicts, but raises new questions:
• Failover: if the leader dies, followers must elect a new one, and everyone must agree on who it is.
• Split brain: a network partition can leave two nodes each believing they are the leader, accepting different writes.
• Lost writes: a write acknowledged by the old leader but not yet replicated can vanish after failover.
Split brain is the two generals problem in disguise: nodes cannot be sure whether the leader is dead or merely unreachable. Real systems use consensus protocols such as Paxos or Raft, or a coordination service such as ZooKeeper, to elect leaders safely.

7. Consistency tradeoffs
You cannot have perfect consistency, perfect availability, and low latency at once in a distributed system. Each design picks a point on the spectrum based on what the business can tolerate.

Strong consistency
Eventual consistency
Reads
Always see the latest write
May see stale data for a short time
Latency
Higher (nodes coordinate)
Lower (respond locally)
Availability
Can refuse requests during failures
Keeps serving during failures
Typical use
Payments, inventory, bookings
Feeds, likes, view counts, caches
The question to ask for each feature is: what is the cost of showing stale data? A wrong bank balance is a serious problem. A like count that is a few seconds behind is not.

8. Two phase commit (2PC)
Two phase commit gets several nodes to commit a transaction together or abort it together. One node acts as the coordinator and the others are participants. It answers the multi-shard problem from section 3.
Phase 1: prepare (voting)
1. The coordinator sends "prepare" to every participant.
2. Each participant does the work, writes it to a durable log, locks the affected data, and replies "yes" or "no."
Phase 2: commit (decision)
1. If every participant voted yes, the coordinator logs the decision and sends "commit." Participants apply the change, release locks, and acknowledge.
2. If any participant voted no or timed out, the coordinator sends "abort" and everyone rolls back.
Example: transferring money from X on node 1 to Y on node 2. Both nodes lock their rows and vote yes, then the coordinator tells both to commit. If node 2 votes no, node 1 rolls back its debit.
Drawbacks
• Blocking: a participant that voted yes must hold its locks until it hears the decision. If the coordinator crashes at that point, participants are stuck.
• Coordinator is a single point of failure unless it is itself replicated.
• Slow: at least two network round trips, plus locks held across them, so throughput suffers.
• Reduced availability: any one node being down aborts or stalls the transaction.
Because of these costs, 2PC is used where strong consistency is essential and is avoided on high-scale, latency-sensitive paths.

9. Eventual consistency
Eventual consistency promises that if no new writes arrive, all replicas will converge to the same value after some time. Until then, different readers may see different values.
A write is acknowledged by one node, then copied to the others in the background. The system stays fast and available, even when nodes or links fail, because nobody waits for agreement.
What you have to handle in return:
• Stale reads: a user may not see their own update right away. Fixes include reading from the leader after a write, or sticky sessions.
• Conflicts: concurrent writes to the same key need a resolution rule, such as last write wins (timestamps), vector clocks, or merge logic like CRDTs.
• Repair: replicas catch up through background sync, read repair, or anti-entropy processes.
Good fits are social feeds, like and view counts, product catalogs, shopping carts, and DNS. Poor fits are balances, stock levels, and seat bookings, where stale data causes real harm. Systems such as Cassandra and DynamoDB are built around this model.

10. Quick revision
Each step of scaling out fixes one problem and introduces the next:
Step
Fixes
New problem
Typical answer
Single node
Simplicity, consistency
Failure, throughput, storage limits
Scale out
Split data (sharding)
Throughput, storage
Multi-shard atomicity, no fault tolerance
Two phase commit; replicas
Copy data (replication)
Fault tolerance, read scale
Stale reads, conflicting writes
Leader per data, sync or async copies
Leader assignment
Write conflicts
Failover, split brain
Consensus (Paxos, Raft)
Strong consistency (2PC)
Correct, atomic updates
Blocking, latency, low availability
Use only where required
Eventual consistency
Speed, availability
Stale data, conflicts
Conflict resolution, repair
Interview takeaways
• Always state the consistency requirement first: what happens if a user sees stale data?
• Pick strong consistency for money and inventory, eventual consistency for feeds and counters.
• Name the cost of whichever you choose: latency and availability for strong, staleness and conflicts for eventual.
• Mention the two generals problem when asked why distributed agreement is hard.