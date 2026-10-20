# How Databases Scale Writes: The Power of the Log (HLD Notes)

## 1. Request condensing

A write heavy system (logging, metrics, chat, IoT sensors, clickstreams) receives huge numbers of small writes. If each one is applied straight to its final place on disk, the database spends most of its time seeking to scattered locations. Disk is far faster at sequential writes than random ones.

Request condensing means batching many incoming writes together and handling them as one group, instead of touching the disk once per request.

- **Fewer disk operations:** one large write replaces many small ones.
- **Sequential access:** the batch is written in a single run, which is the disk's best case.
- **Higher throughput:** the database can absorb bursts without falling behind.

The trade is that a batch held only in memory could be lost in a crash, which is why it is paired with a log on disk. The rest of these notes build the design: an append-only log, then sorted files, then merging and filtering.

## 2. The log as a linked list

The simplest write-friendly structure is a log: every write is appended to the end of a file, and nothing is ever changed in place. Think of it as a linked list where each new record is added at the tail.

**Why writes are fast**

- An append needs no search: just write at the end.
- It is sequential disk I/O, which is the fastest kind.
- Updates and deletes are also appends: a new record with the new value, or a "deleted" marker.

**Why reads are slow**

- To find a key you must scan the log, usually from the newest entry backwards, until you find it. That is O(n).
- The log keeps growing, including old versions of the same key, so both space and read time get worse.

So a plain log gives O(1) writes but O(n) reads. The question becomes how to keep the write speed and fix the reads.

## 3. B+ tree vs linked list log

Traditional databases use a B+ tree, which keeps data sorted in fixed-size pages. A lookup walks a few levels down the tree, so reads are fast. Writes are the weak side: each insert or update finds the right page, and that page is rewritten in place. That is a random disk write, and a full page may have to split.

|  | B+ tree | Append-only log |
| --- | --- | --- |
| Write | O(log n), random disk writes, page splits | O(1), sequential append |
| Read by key | O(log n), few disk reads | O(n), scan the log |
| Range scan | Fast, data is sorted | Slow |
| Write amplification | Whole page rewritten for a small change | Low |
| Best for | Read heavy workloads | Write heavy workloads |

Neither is ideal alone. A Log Structured Merge Tree (LSM tree) combines them: take the fast append-style writes of the log, and add sorted files plus background merging to recover good read speed. Databases built this way include Cassandra, HBase, LevelDB, and RocksDB.

## 4. Sorted String Table (SSTable)

An SSTable is an immutable file on disk that stores key-value pairs sorted by key. Sorting is what makes reads fast again.

**The write path**

1. Each write is first appended to a write-ahead log (WAL) on disk, so it survives a crash.
2. The write is also inserted into the memtable, an in-memory sorted structure (usually a balanced tree or skip list).
3. When the memtable reaches a size limit, it is flushed to disk as a new SSTable. This is one sequential write of already-sorted data.
4. The WAL for that data is discarded, and a fresh memtable starts.

**The read path**

1. Check the memtable first, since it holds the newest data.
2. If the key is not there, check SSTables from newest to oldest and return the first match.
3. Inside one SSTable, binary search on a sparse index finds the key without scanning the whole file.

**Why this works:** writes are only appends and sequential flushes, while reads can binary search sorted files. SSTables are never edited, which makes them simple, safe to cache, and easy to back up.

**The catch:** over time many SSTables pile up, and a read may have to check several of them. Old versions of keys and deleted data waste space. That is solved by compaction.

## 5. Compaction

Compaction is a background process that merges several SSTables into fewer, larger ones. Because every SSTable is already sorted, merging works like the merge step of merge sort: read the files side by side and write one sorted output file in a single sequential pass.

**What it does during the merge**

- Keeps only the newest value when the same key appears in several files.
- Drops deleted keys. A delete is stored as a tombstone marker, and compaction removes both the marker and the old data once it is safe to do so.
- Reduces the number of files a read must check.
- Frees disk space, then deletes the old input files.

**Common strategies**

| Strategy | How it works | Good for |
| --- | --- | --- |
| Size-tiered | Merge SSTables of similar size together | Write heavy loads, less write amplification |
| Leveled | Files organised into levels with non-overlapping key ranges; merge into the next level | Read heavy loads, fewer files to check |

**Costs:** compaction uses disk and CPU in the background and rewrites data several times (write amplification). It is the price paid for fast writes and acceptable reads.

## 6. Bloom filters

A read for a key that does not exist is the worst case: the database checks every SSTable before it can say "not found." A Bloom filter avoids most of that disk work.

A Bloom filter is a compact, in-memory structure that answers "is this key possibly in this SSTable?"

- **"Definitely not present":** the answer is always right, so the SSTable is skipped with no disk read.
- **"Possibly present":** the answer can be wrong (a false positive), so the SSTable is then read to confirm.
- **No false negatives:** a key that is present is never reported as absent.

**How it works**

1. Keep a bit array of m bits, all set to 0, and k different hash functions.
2. To add a key, hash it k times and set those k bit positions to 1.
3. To check a key, hash it k times. If any of those bits is 0, the key is definitely absent. If all are 1, it is possibly present.

Each SSTable gets its own filter, kept in memory. More bits per key lowers the false positive rate, at the cost of memory. The result is that most lookups touch only the few SSTables that really might hold the key.

Bloom filters cannot delete keys and cannot list contents, but for this job they are a cheap and effective shortcut.

## 7. Quick revision

| Piece | Role in the LSM tree | Fixes |
| --- | --- | --- |
| Write-ahead log | Sequential append of every write | Crash safety |
| Memtable | Sorted in-memory buffer of recent writes | Fast writes, batching |
| SSTable | Immutable sorted file flushed from the memtable | Fast sequential writes, binary search reads |
| Compaction | Background merge of SSTables | Too many files, duplicates, tombstones |
| Bloom filter | In-memory check before reading an SSTable | Wasted disk reads for missing keys |

|  | LSM tree | B+ tree |
| --- | --- | --- |
| Write speed | Very high, sequential | Moderate, random in-place |
| Read speed | Good, but may check several files | Very good, few page reads |
| Space and write amplification | Compaction rewrites data | Page rewrites and splits |
| Examples | Cassandra, HBase, RocksDB, LevelDB | MySQL InnoDB, PostgreSQL |

**Interview takeaways**

- State the problem: random in-place writes are slow, sequential appends are fast.
- Walk the path: WAL, memtable, flush to SSTable, background compaction.
- Explain how reads are rescued: sorted files, newest-first lookup, and Bloom filters.
- Name the tradeoff: LSM trees favour writes and cost extra reads and background compaction work; B+ trees favour reads.
