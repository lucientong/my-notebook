# MySQL In Depth

Language: English | [中文](../后端知识库/02-MySQL深入.md)

---

## Table of Contents

1. [Storage Engines](#1-storage-engines)
2. [Logging Mechanism](#2-logging-mechanism)
3. [Indexing Principles](#3-indexing-principles)
4. [SQL Execution](#4-sql-execution)
5. [Transactions and Locks](#5-transactions-and-locks)
6. [Replication and Consistency](#6-replication-and-consistency)
7. [Performance Optimization](#7-performance-optimization)
8. [Data Modeling](#8-data-modeling)
9. [Practical Cases](#9-practical-cases)
10. [Interview Self-Check](#10-interview-self-check)

---

## 1. Storage Engines

### 1.1 InnoDB vs MyISAM ⭐⭐⭐

In modern MySQL, InnoDB is the default and production-oriented storage engine.

| Dimension | InnoDB | MyISAM |
|-----------|--------|--------|
| Transactions | yes | no |
| Row-level lock | yes | no, table-level lock |
| Foreign key | yes | no |
| Crash recovery | strong, redo log based | weak |
| MVCC | yes | no |
| Typical use | OLTP systems | legacy read-heavy tables |

Interview answer:

> InnoDB is preferred for most backend systems because it supports transactions, MVCC, row-level locking, crash recovery, and better concurrency. MyISAM is mostly legacy and lacks transaction safety.

### 1.2 InnoDB Architecture

Key components:

- Buffer pool: caches data pages and index pages.
- Redo log: crash recovery.
- Undo log: rollback and MVCC.
- Change buffer: optimizes secondary index writes.
- Adaptive hash index: accelerates hot point lookups.
- Doublewrite buffer: prevents partial page writes.

---

## 2. Logging Mechanism

### 2.1 Redo Log, Undo Log, and Binlog ⭐⭐⭐

| Log | Layer | Purpose |
|-----|-------|---------|
| Redo log | InnoDB | crash recovery, physical log |
| Undo log | InnoDB | rollback and MVCC snapshots |
| Binlog | MySQL server | replication and point-in-time recovery |

Commit flow is commonly described as a two-phase commit between redo log and binlog:

```text
1. Write redo log in prepare state.
2. Write binlog.
3. Commit redo log.
```

This keeps transactional storage and replication log consistent.

### 2.2 WAL

Write-Ahead Logging means log records are flushed before dirty pages are written back.

**Dirty page**: a Buffer Pool page that has been modified in memory but not yet written back to the `.ibd` data file. The in-memory copy is newer than disk; that is fine as long as redo is durable.

**Commit vs flush order** (with `innodb_flush_log_at_trx_commit=1`):

```text
UPDATE ...     → modify Buffer Pool page (becomes dirty) + write redo buffer
COMMIT
  → fsync redo log   ← this is inside COMMIT; success is returned after redo is durable
  → return OK to client
... later ...
background         → flush dirty pages to .ibd (async; not required for COMMIT success)
```

Common misconception: fsync of redo is **not** “after the transaction already finished”; it **is** the durability step of COMMIT. Flushing dirty pages **is** after COMMIT and can lag far behind.

Benefits:

- Random page writes are converted into sequential log writes.
- Crash recovery can replay redo logs.
- Dirty pages can be flushed lazily.

---

## 3. Indexing Principles

### 3.1 B+Tree Structure

MySQL **InnoDB** indexes are usually B+Trees. Not every engine/index type is:

| Engine | Typical index | Where row data lives |
|--------|---------------|----------------------|
| InnoDB | B+Tree | Primary-key (clustered) leaf nodes |
| MyISAM | B+Tree | Separate `.MYD` file; index leaves store row pointers |
| MEMORY | HASH by default (or BTree) | In-memory table |

People saying “MySQL uses B+Trees” usually mean **InnoDB’s default indexes**.

Reasons InnoDB uses B+Trees:

- High fan-out reduces tree height.
- Leaf nodes are ordered and linked, good for range scans.
- Internal nodes only store keys and child pointers, improving cache/page efficiency.

```text
Root
 ├── Internal page
 │    ├── Leaf page -> Leaf page -> Leaf page
```

### 3.2 Clustered and Secondary Indexes

One InnoDB table is **multiple B+Trees**, not one:

```text
Tree 1 — PRIMARY (clustered): leaves hold full rows, ordered by PK
Tree 2 — idx_name:            leaves hold (name, id) only
Tree 3 — idx_email:           leaves hold (email, id) only
...
```

So: **full row data exists only in the primary-key tree’s leaves**. Secondary trees do not duplicate the whole row; they store secondary key + primary key, then **back to the PK tree** when other columns are needed.

InnoDB primary key index is clustered:

- Leaf node stores the full row.
- Table data is organized by primary key.

Secondary index:

- Leaf node stores secondary key plus primary key.
- If selected columns are not covered, MySQL performs a back-to-table lookup by primary key.

**Unique secondary index and back-to-table lookup**

A non-primary unique index is still a secondary index: the leaf stores `(unique_key, primary_key)`, not the full row. Uniqueness does **not** avoid table lookup.

| Query | Lookup? |
|-------|---------|
| `SELECT * FROM users WHERE email = ?` on unique `email` | Yes — needs columns not in the index |
| `SELECT id, email FROM users WHERE email = ?` | No — covering |
| `SELECT * FROM users WHERE id = ?` (PK) | No — clustered leaf is the row |

Difference vs normal secondary index: unique equality matches **at most one row** (at most one lookup); a non-unique key may match many rows and multiply lookups. Cost of one lookup is usually fine; thousands of random PK lookups are expensive. Prefer covering indexes and avoid `SELECT *` on large ranges.

### 3.3 Composite Indexes

Leftmost prefix rule:

```sql
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at);
```

This index can support:

- `WHERE user_id = ?`
- `WHERE user_id = ? AND status = ?`
- `WHERE user_id = ? AND status = ? AND created_at > ?`

It cannot efficiently support `WHERE status = ?` alone.

### 3.4 Covering Index and Back-to-Table Lookup

Covering index means all required columns are in the index.

```sql
SELECT user_id, status
FROM orders
WHERE user_id = ?;
```

If `(user_id, status)` covers the query, no row lookup is needed.

### 3.5 Index Failure Cases

Indexes may not be used effectively when:

- Function is applied to indexed column.
- Leading wildcard is used: `LIKE '%abc'`.
- Implicit type conversion happens.
- OR conditions are not all index-friendly.
- Low selectivity makes full scan cheaper.
- Composite index violates leftmost prefix.

### 3.6 Index Condition Pushdown

Index Condition Pushdown lets storage engine filter more conditions at index scan time, reducing row lookups.

---

## 4. SQL Execution

### 4.1 Lifecycle of a SQL Query

```text
Client
-> connection/authentication
-> parser
-> preprocessor
-> optimizer
-> executor
-> storage engine
```

### 4.2 Optimizer

The optimizer chooses execution plans based on:

- Index statistics.
- Cardinality.
- Cost model.
- Join order.
- Predicate selectivity.

### 4.3 EXPLAIN

Important fields:

| Field | Meaning |
|-------|---------|
| `type` | access type, such as `const`, `ref`, `range`, `ALL` |
| `key` | selected index |
| `rows` | estimated scanned rows |
| `Extra` | extra operations, such as filesort or temporary |

Red flags:

- `type = ALL` on large table.
- `Using filesort`.
- `Using temporary`.
- very high `rows`.

### 4.4 Prepared Statements

Prepared statements reduce parse overhead and avoid SQL injection when used correctly.

---

## 5. Transactions and Locks

### 5.1 Isolation Levels

| Isolation | Problems Prevented |
|-----------|--------------------|
| Read Uncommitted | almost none |
| Read Committed | dirty reads |
| Repeatable Read | dirty/non-repeatable reads |
| Serializable | phantom reads by serialization |

InnoDB default isolation is Repeatable Read. It uses MVCC plus next-key locks to address many phantom scenarios.

### 5.2 MVCC ⭐⭐⭐

MVCC enables consistent reads without blocking writes.

Key concepts:

- Hidden transaction ID (`DB_TRX_ID`).
- Roll pointer to undo log (`DB_ROLL_PTR`).
- Read view + active transaction list.
- Version chain walk until the first **visible** version.

**Essence (common misconception)**

1. Starts from the current row version.
2. Applies Read View visibility rules.
3. Walks the undo chain until it finds the first visible version.

| Concern | MVCC snapshot read |
|---------|-------------------|
| Dirty read | Prevented in RC/RR (other txs' uncommitted versions are invisible) |
| Non-repeatable read | Prevented mainly in **RR** (reuse one Read View); **RC** creates a new Read View per SELECT and can still see others' commits |
| Phantom read | Snapshot SELECT mostly filters invisible inserts; current reads (`SELECT FOR UPDATE` / `UPDATE`) need next-key locks |

Also: a transaction always sees its **own** uncommitted changes. In InnoDB RR, the Read View is typically created on the **first consistent SELECT**, not strictly at `BEGIN`.

At Repeatable Read, the same transaction usually reuses the same read view for consistent reads, so repeated reads see a stable snapshot.

### 5.3 Lock Types

Common locks:

- Shared lock and exclusive lock.
- Record lock.
- Gap lock.
- Next-key lock.
- Intention lock.
- Insert intention lock.

Next-key lock = record lock + gap lock. It protects index ranges and helps prevent phantom inserts.

### 5.4 Deadlock Diagnosis

Useful commands:

```sql
SHOW ENGINE INNODB STATUS;
```

Practical prevention:

- Access tables and rows in consistent order.
- Keep transactions short.
- Add proper indexes to avoid locking too many rows.
- Avoid user interaction or remote calls inside transactions.

---

## 6. Replication and Consistency

### 6.1 Master-Replica Replication

Classic replication flow:

```text
Primary writes binlog
-> replica IO thread pulls binlog
-> relay log
-> SQL thread applies changes
```

**Async vs semi-sync — two commit-wait policies, not two stacked features**

| Mode | When primary returns COMMIT OK | Notes |
|------|--------------------------------|-------|
| **Async** (default) | After local binlog/engine commit; **does not wait** for replicas | Replica lag can mean committed txs never reached any replica if primary dies |
| **Semi-sync** (plugin) | Waits for **≥1** replica ACK that binlog is in **relay log** | Guarantees log receipt, **not** that SQL apply finished; may fall back to async on ACK timeout |

Same binlog → IO → relay → SQL pipeline; semi-sync only adds “wait for ACK before return OK.”

Replication formats:

- Statement-based (SQL text; risky with nondeterministic functions).
- Row-based (row images; safest; common for DTS/CDC).
- Mixed.

Row-based replication is more deterministic and common.

**What may skip binlog or still diverge**

| Case | Effect |
|------|--------|
| `log_bin` off / `SET sql_log_bin=0` | Change never reaches replicas |
| `binlog_do_db` / `binlog_ignore_db` | Master may not log selected DBs |
| Direct writes on replica / raw `.ibd` copy | Diverges outside replication |
| STATEMENT + `NOW()`/`UUID()`/`RAND()` | Logged but results differ |
| `replicate_*` filters / skip counter / skip GTID | Logged but not applied (or skipped) on replica |

Prefer ROW, keep replicas read-only, avoid session `sql_log_bin=0` in app paths, checksum periodically.

Replica lag causes: large transactions, slow apply (historically single SQL thread), weak replica hardware, long queries holding locks. Mitigations: `slave_parallel_workers`, smaller transactions.

### 6.2 Read-After-Write Consistency

Common strategies:

- Read from primary after write.
- Route users to same replica after checking delay.
- Use GTID / log position wait.
- Cache recently written keys and force primary reads.

---

## 7. Performance Optimization

### 7.1 Slow Query Optimization

Workflow:

```text
find slow SQL -> EXPLAIN -> check indexes -> check rows scanned -> rewrite query -> verify
```

Checklist:

- Avoid `SELECT *`.
- Add indexes for high-selectivity predicates and join keys.
- Avoid deep offset pagination.
- Avoid large transactions.
- Watch temporary tables and filesort.

### 7.2 Batch Operations

Batch writes reduce round trips but should be bounded to avoid long locks and huge transactions.

```sql
INSERT INTO orders(id, user_id, amount)
VALUES (?, ?, ?), (?, ?, ?), (?, ?, ?);
```

### 7.3 Deep Pagination

Avoid:

```sql
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
```

Prefer keyset pagination:

```sql
SELECT * FROM orders
WHERE id > ?
ORDER BY id
LIMIT 20;
```

---

## 8. Data Modeling

### 8.1 Normalization and Denormalization

Normalization reduces redundancy and update anomalies. Denormalization improves read performance by duplicating data intentionally.

Use denormalization when:

- Read traffic is much heavier than writes.
- Join cost is high.
- Data consistency can be maintained by async repair or clear ownership.

### 8.2 Sharding

Sharding is used when a single database cannot handle data volume or write throughput.

Challenges:

- Distributed transactions.
- Cross-shard queries.
- Rebalancing.
- Global unique IDs.
- Hot shards.

Shard key selection matters more than the sharding middleware.

---

## 9. Practical Cases

### 9.1 Financial Transaction Table

Design principles:

- Use immutable ledger entries.
- Use unique request ID for idempotency.
- Use transaction status state machine.
- Keep audit fields.
- Use database constraints as correctness boundary.

### 9.2 Game Leaderboard

Possible design:

- MySQL stores durable ranking records.
- Redis ZSet serves hot leaderboard reads.
- Async jobs reconcile Redis and MySQL.
- Use partitioning by season or region.

---

## 10. Interview Self-Check

### Q1: Why does InnoDB use B+Tree instead of hash index?

**Answer:** B+Tree supports range scans, ordering, and stable disk/page access with high fan-out. Hash indexes are good for equality lookup but poor for range queries and ordering.

### Q2: What is MVCC?

**Answer:** MVCC provides snapshot reads through transaction versions, undo logs, and read views. It allows readers and writers to avoid blocking each other in many cases.

### Q3: What is the difference between redo log and binlog?

**Answer:** Redo log is InnoDB's physical crash recovery log. Binlog is the MySQL server-level logical replication and PITR log.

### Q4: What is a covering index?

**Answer:** A covering index contains all columns needed by the query, so MySQL can answer from the index without fetching full rows.

### Q5: How do you optimize a slow SQL query?

**Answer:** Capture the SQL, run `EXPLAIN`, inspect access type/index/rows/Extra, add or adjust indexes, rewrite predicates, reduce returned columns, avoid deep pagination, and verify with real data.

### Q6: What causes deadlocks?

**Answer:** Different transactions acquire locks in conflicting order. Missing indexes can enlarge lock ranges and increase deadlock probability.

### Q7: How do you handle read-after-write consistency in primary-replica architecture?

**Answer:** Route recent reads to primary, wait for replica to catch up by GTID/binlog position, or design session-level consistency policies.

### Q8: What is next-key lock?

**Answer:** A next-key lock locks an index record and the gap before it. It helps prevent phantom inserts under Repeatable Read.

### Q9: How do you design idempotent order creation?

**Answer:** Require idempotency key, store it with a unique constraint, wrap creation in a transaction, and return the existing result for duplicate requests.

### Q10: When should you shard MySQL?

**Answer:** Only after schema, indexes, queries, caching, and vertical scaling are insufficient. Sharding adds operational and consistency complexity.
