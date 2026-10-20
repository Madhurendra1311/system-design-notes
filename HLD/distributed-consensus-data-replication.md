# Master Slave Replication and Distributed Consensus (HLD Notes)

## 1. Problem statement

A single database server is a single point of failure: if it crashes or its disk is lost, the application goes down and data may be gone for good. Backups help, but restoring from a backup is slow and loses everything written since the last one.

We want a design that gives:

- **Reliability:** the system keeps working if a machine fails.
- **Backups:** an up-to-date copy of the data exists somewhere else.
- **Read scale:** more machines can serve read traffic.

The standard answer is replication: keep copies of the data on several machines. The rest of these notes cover how to do that, and the consistency and availability tradeoffs it forces.

## 2. Replication (master slave)

In master slave replication, one node (the master, also called primary or leader) accepts all writes. One or more slaves (replicas, followers) receive a copy of every change and keep their own copy of the data.

**How it flows**

1. The client sends a write to the master.
2. The master applies it and records it in a replication log.
3. The master ships the log entries to each slave.
4. Each slave applies the entries in the same order, so it ends up with the same data.

**What each role does**

- **Master:** the single place for writes, which avoids conflicting updates.
- **Slaves:** serve read traffic, act as live backups, and can be promoted to master if the master fails.

**Benefits:** fault tolerance, read scaling, and a copy of the data for backups and analytics without loading the master.

**Costs:** writes still hit one machine, so write throughput does not scale, and a slave can lag behind the master.

## 3. Synchronous vs asynchronous replication

The key choice is when the master tells the client "write done."

|  | Synchronous | Asynchronous |
| --- | --- | --- |
| When the client gets its ack | After the slave(s) confirm the write | Right after the master writes locally |
| Consistency | Strong: slaves match the master | Eventual: slaves may lag |
| Write latency | Higher, waits on the network and slowest slave | Low |
| If the master crashes | No acknowledged write is lost | Recent acknowledged writes can be lost |
| If a slave is down | Writes may block or fail | Writes continue unaffected |
| Availability | Lower | Higher |

**Semi-synchronous** is the common middle path: wait for one slave to confirm and let the rest catch up in the background. You keep at least one fully up-to-date copy without waiting on every replica.

**Rule of thumb:** choose synchronous when losing a write is unacceptable (payments, orders), and asynchronous when speed and uptime matter more than seeing the very latest data (feeds, analytics replicas).

This is the consistency versus availability tradeoff that interviewers look for.

## 4. Peer to peer data transfer

If the master alone ships data to every slave, it becomes a bottleneck: its network and CPU are spent on copying instead of serving writes. With many slaves, replication lag grows.

In peer to peer transfer, slaves also pass data to each other.

- **Chained replication:** the master sends to slave 1, slave 1 forwards to slave 2, and so on. The master only feeds one node, so its load stays flat as slaves are added.
- **Gossip or tree style:** each node that has the data shares it with a few peers, spreading it quickly across a large cluster.

**Tradeoffs**

- Master load drops and the system scales to many replicas.
- Distant replicas receive data later, since it passes through more hops.
- If a middle node fails, everything downstream stops receiving updates until the chain is repaired.

A related design is multi-master (peer to peer for writes too), where several nodes accept writes. It removes the single write bottleneck but brings back conflicting updates that must be resolved.

## 5. Split brain problem

Split brain happens when two nodes both believe they are the master and both accept writes. Their data then diverges, and merging it later can be painful or impossible.

**How it happens**

1. A network partition (or a slow, overloaded master) stops the slaves from hearing the master's heartbeat.
2. The slaves assume the master is dead and promote one of them.
3. The old master is actually alive. Clients on its side of the partition keep writing to it.
4. Now two masters hold different data.

**Why it is dangerous:** two accounts could each be debited from different masters, or two users could be sold the last seat. After the partition heals, the histories conflict.

**How to prevent it**

- **Quorum (majority) rule:** a node may act as master only if it can reach a majority of the cluster. The minority side of a partition steps down and stops accepting writes.
- **Fencing:** every new master gets a higher epoch or token number. Storage and replicas reject writes carrying an older token, so a stale master is cut off.
- **Odd number of nodes or a witness node:** avoids ties when the cluster splits evenly.
- **Reliable failure detection:** sensible timeouts so a slow master is not declared dead too quickly.

The cost of these safeguards is availability: the minority side of a partition cannot take writes until it rejoins.

## 6. Distributed consensus

Consensus means the components of a system agree on one value, even when messages are delayed or nodes fail. Master slave replication depends on it: the nodes must agree on who the master is, and on which writes have been committed. It shows up in leader election, distributed transactions, and replicated logs.

Four popular techniques:

**Two phase commit (2PC).** A coordinator asks all participants to prepare, and only if all vote yes tells them to commit; otherwise everyone aborts. It gives atomic commits across nodes, but participants block if the coordinator fails, and it is slow.

**Multi Version Concurrency Control (MVCC).** Instead of overwriting data, the database keeps several versions of each row, each stamped with a transaction id or timestamp. A reader sees a consistent snapshot as of its start time, so readers do not block writers and writers do not block readers. Used by PostgreSQL and MySQL InnoDB. It solves concurrent access on a node more than agreement between nodes, but is a core tool for consistent reads on replicas.

**SAGA.** A long transaction is split into a series of local transactions, one per service. If a step fails, the saga runs compensating actions to undo the earlier steps (for example, refund a payment if shipping fails). It avoids holding locks across services and scales well, but the system is only eventually consistent and compensations must be designed carefully.

**Quorum.** With N replicas, a write must be confirmed by W of them and a read must consult R of them. If R + W > N, every read overlaps with the latest write, so it sees the newest value. Majority quorums (more than half) are also how a leader is elected safely. Raft and Paxos are consensus protocols built on this idea.

| Technique | Gives you | Main cost |
| --- | --- | --- |
| 2PC | Atomic commit across nodes | Blocking, latency |
| MVCC | Consistent snapshots without read locks | Storage for old versions, cleanup |
| SAGA | Cross-service workflows without locks | Eventual consistency, compensation logic |
| Quorum | Tunable consistency, safe leader election | More replicas to contact per operation |

## 7. Quick revision

| Topic | One-line summary | Watch out for |
| --- | --- | --- |
| Master slave | One node takes writes, others copy and serve reads | Write scaling, replica lag |
| Synchronous | Ack after replicas confirm | Latency, blocked writes |
| Asynchronous | Ack immediately, copy later | Stale reads, lost recent writes on failover |
| Peer to peer transfer | Replicas pass data to each other | Longer lag, broken chains |
| Split brain | Two masters after a partition | Diverging data |
| Quorum | Majority agreement, R + W > N | Fewer nodes reachable means no writes |
| 2PC / SAGA / MVCC | Atomic commit / compensating steps / versioned snapshots | Blocking / eventual consistency / storage |

**Interview takeaways**

- Open with the goal: reliability, backups, and read scale through replication.
- Name the consistency versus availability tradeoff whenever you pick sync or async.
- Always mention how you handle master failure: failure detection, election by majority, and fencing.
- Pick the consensus tool by the problem: 2PC for atomic commits, SAGA for cross-service workflows, quorum for leader election and tunable reads, MVCC for snapshot reads.
