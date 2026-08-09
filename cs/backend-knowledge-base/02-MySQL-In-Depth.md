# MySQL In Depth

Language: English | [中文](../后端知识库/02-MySQL深入.md)

> **Baseline**: MySQL **8.0** unless noted. Call out 5.7 vs 8.0 differences (redo capacity, REPLICA/SOURCE terms, descending indexes, etc.).

---

## Table of Contents

1. [Storage Engines](#1-storage-engines)
2. [Logging Mechanism](#2-logging-mechanism)
3. [InnoDB Memory Subsystem](#3-innodb-memory-subsystem)
4. [Indexing Principles](#4-indexing-principles)
5. [SQL Execution](#5-sql-execution)
6. [Transactions and Locks](#6-transactions-and-locks)
7. [Replication and Consistency](#7-replication-and-consistency)
8. [Online DDL, Backup, 8.0 Features](#8-online-ddl-backup-80-features)
9. [Performance Optimization](#9-performance-optimization)
10. [Data Modeling](#10-data-modeling)
11. [Practical Cases](#11-practical-cases)
12. [Interview Self-Check](#12-interview-self-check)

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

**Dual-1 durability**: production money-path often sets `innodb_flush_log_at_trx_commit=1` **and** `sync_binlog=1`. Changing only one still risks lost commits or primary/replica gaps. Relax both together only with an explicit RPO budget.

Redo capacity: before 8.0.30, classic default was two `ib_logfile*` (often 48MB each); 8.0.30+ uses `innodb_redo_log_capacity` (default 100MB total). Server default `binlog_format` has been **ROW** since 5.7.7.

---

## 3. InnoDB Memory Subsystem

### 3.1 Buffer Pool ⭐⭐⭐

Hot pages live in the Buffer Pool. Reads hit memory when possible; writes modify in-memory pages (dirty) and write redo, then flush later.

**Why not a plain LRU?** Full scans would push hot pages out. InnoDB uses **midpoint insertion**: new pages enter the *old* sublist; only pages accessed again after `innodb_old_blocks_time` promote to the *new* sublist. Scan-once pages die in the old zone without polluting hot data.

Related lists: LRU (eviction order), Free (empty pages), Flush (dirty pages ordered by LSN for checkpointing).

### 3.2 Change Buffer / Doublewrite / Purge

- **Change Buffer**: defer changes to **non-unique secondary** index pages that are not in memory (unique indexes usually cannot, because uniqueness must be checked immediately).
- **Doublewrite**: write a full page copy sequentially before the real page write to survive torn pages.
- **Purge**: after commit, old undo versions stay until no Read View needs them. Long transactions block purge → undo bloat and slower version-chain walks.

---

## 4. Indexing Principles

### 4.1 B+Tree Structure

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

### 4.2 Clustered and Secondary Indexes

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

### 4.3 Composite Indexes

Order columns by **query shape**, not “highest cardinality first.” A low-cardinality equality column can still lead if every query filters it first.

Leftmost prefix rule:

```sql
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at);
```

This index can support:

- `WHERE user_id = ?`
- `WHERE user_id = ? AND status = ?`
- `WHERE user_id = ? AND status = ? AND created_at > ?`

It cannot efficiently support `WHERE status = ?` alone.

### 4.4 Covering Index and Back-to-Table Lookup

Covering index means all required columns are in the index.

```sql
SELECT user_id, status
FROM orders
WHERE user_id = ?;
```

If `(user_id, status)` covers the query, no row lookup is needed.

### 4.5 Index Failure Cases

Indexes may not be used effectively when:

- Function is applied to indexed column.
- Leading wildcard is used: `LIKE '%abc'`.
- Implicit type conversion happens.
- OR conditions are not all index-friendly.
- Low selectivity makes full scan cheaper.
- Composite index violates leftmost prefix.

### 4.6 Index Condition Pushdown

Index Condition Pushdown lets storage engine filter more conditions at index scan time, reducing row lookups.

---

## 5. SQL Execution

### 5.1 Lifecycle of a SQL Query

```text
Client
-> connection/authentication
-> parser
-> preprocessor
-> optimizer
-> executor
-> storage engine
```

### 5.2 Optimizer

The optimizer chooses execution plans based on:

- Index statistics.
- Cardinality.
- Cost model.
- Join order.
- Predicate selectivity.

### 5.3 EXPLAIN

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

### 5.4 Prepared Statements

Prepared statements reduce parse overhead and avoid SQL injection when used correctly.

---

## 6. Transactions and Locks

### 6.1 Isolation Levels

| Isolation | Problems Prevented |
|-----------|--------------------|
| Read Uncommitted | almost none |
| Read Committed | dirty reads |
| Repeatable Read | dirty/non-repeatable reads (snapshot path) |
| Serializable | strongest; more locking |

InnoDB default is Repeatable Read. **RR ≠ “always next-key on every SELECT”**: plain `SELECT` is a snapshot read (MVCC); `SELECT FOR UPDATE` / `UPDATE` / `DELETE` are current reads (locks).

### 6.2 MVCC ⭐⭐⭐

Key concepts: `DB_TRX_ID`, `DB_ROLL_PTR`, Read View, undo version chain.

Walk from the **current** version until the first **visible** one. Inserts also have `trx_id` — MVCC can hide them on snapshot reads; it does **not** mean “MVCC cannot handle INSERT.”

| Concern | Snapshot read |
|---------|----------------|
| Dirty read | Blocked in RC/RR |
| Non-repeatable read | RR reuses one Read View; RC makes a new one per SELECT |
| Phantom | Snapshot mostly filters; current reads need next-key |

Read View in RR is created on the **first consistent SELECT**, not at `BEGIN`. A late first SELECT can see commits that happened after `BEGIN`.

### 6.3 Lock Types and Rules (RR)

- Intention locks (IS/IX) on the table before row locks — IX is compatible with IX.
- Insert intention lock: concurrent inserts into the same gap with different values do not block each other.
- AUTO-INC modes: 8.0 default interleaved (best concurrency; values may have gaps).
- `FOR SHARE` (8.0; formerly `LOCK IN SHARE MODE`), `FOR UPDATE NOWAIT` / `SKIP LOCKED`.
- **MDL**: DML holds metadata read locks; a stuck query can block `ALTER` and then block the whole table.

| Scenario | Locks |
|----------|-------|
| Unique equality, **hit** | **Record lock only** (no gap) |
| Unique equality, **miss** | Gap lock |
| Non-unique equality | Next-key + following gap |
| Range (`BETWEEN`, `>`) | Next-key over scanned range |
| No usable index | Locks scanned rows/gaps (near table-wide) |
| RC | No gap locks (record locks only) |

Common mistake: `WHERE id = 10 FOR UPDATE` on PK does **not** block inserting `id=7`.

Observe waits: `performance_schema.data_locks` / `data_lock_waits`.

### 6.4 Deadlock Diagnosis

```sql
SHOW ENGINE INNODB STATUS\G  -- LATEST DETECTED DEADLOCK
```

Prevent: consistent lock order, short txs, indexes, optional RC to drop gap locks, optimistic versioning when conflicts are rare.

---

## 7. Replication and Consistency

### 7.1 Primary–Replica

```text
Primary binlog → replica IO thread → relay log → SQL/applier threads
```

| Mode | COMMIT returns after | Notes |
|------|----------------------|-------|
| Async | Local durability only | May lose txs on primary crash |
| Semi-sync | ≥1 replica has relay log ACK | Not “apply finished”; may fall back to async |

Prefer **ROW**. Terms: `SHOW REPLICA STATUS`, `SOURCE_*` (old `SLAVE`/`MASTER` names deprecated). Parallel apply: `replica_parallel_workers` (+ LOGICAL_CLOCK / writeset dependency).

**GTID**: `uuid:seq` global IDs; `MASTER_AUTO_POSITION=1` / auto-position; skip a bad event by injecting an empty transaction with that GTID — do not casually `sql_replica_skip_counter` under GTID. Watch `gtid_purged` when restoring backups.

**MGR** (concept): group consensus, single-primary failover, write-set certification; needs PK, ROW, GTID. Prefer single-primary in production.

### 7.2 Read-After-Write

- Force primary for a short window after write.
- `WAIT_FOR_EXECUTED_GTID_SET` (prefer over position wait).
- Session sticky to primary.
- Cache-aside for recently written keys.

---

## 8. Online DDL, Backup, 8.0 Features

### 8.1 Online DDL

Prefer **INSTANT** → **INPLACE** → **COPY**. Large tables: **gh-ost** or **pt-osc**; watch MDL waits and replica lag.

### 8.2 Backup and PITR

Logical (`mysqldump`) for small DBs; physical hot backup (XtraBackup) for large. PITR = full backup + binlog/GTID replay. Rehearse restores; set `gtid_purged` correctly.

### 8.3 MySQL 8.0 Highlights

Atomic DDL, true descending indexes, histograms, `EXPLAIN ANALYZE`, window functions/CTE, invisible indexes, REPLICA/SOURCE naming.

---

## 9. Performance Optimization

### 9.1 Slow-Query Loop

```text
slow log / performance_schema digest
  → pt-query-digest / mysqldumpslow
  → EXPLAIN / EXPLAIN ANALYZE
  → fix SQL/index/schema → regress
```

### 9.2 Batch Operations

Bound batch size to avoid long locks and huge transactions.

### 9.3 Deep Pagination

Avoid large `OFFSET`. Prefer keyset (`WHERE id > ? ORDER BY id LIMIT n`) or deferred join (cover ids first, then join back).

---

## 10. Data Modeling

### 10.1 Normalization and Denormalization

Normalize for write correctness; denormalize when read fan-out dominates and ownership/repair is clear.

### 10.2 Sharding

Last resort after indexes, caching, and vertical scale. Hard parts: cross-shard queries, resharding, global IDs, hot shards. Shard key beats middleware brand.

---

## 11. Practical Cases

### 11.1 Financial Ledger

Immutable entries, unique request ID, state machine, audit columns, DB constraints as safety net; `FOR UPDATE` or version checks for balance updates.

### 11.2 Game Leaderboard

MySQL durable store + Redis ZSet hot path + async reconcile; partition by season/region.

---

## 12. Interview Self-Check

### Q1: Why B+Tree over hash?

**Answer:** Range, order, high fan-out on disk pages. Hash is equality-only.
**Follow-up:** Rough rows in a 3-level tree with 16KB pages?

### Q2: MVCC and Read View — RC vs RR?

**Answer:** Version chain + Read View. RR creates view on **first consistent read** and reuses it; RC creates per SELECT.
**Follow-up:** Can a late first SELECT after BEGIN see commits that happened after BEGIN? Yes.

### Q3: Redo vs binlog? Dual-1?

**Answer:** Redo = InnoDB crash recovery; binlog = replication/PITR. Dual-1 = flush redo + sync binlog on every commit.

### Q4: Buffer Pool midpoint / doublewrite / change buffer?

**Answer:** Midpoint protects hot pages from scans; doublewrite prevents torn pages; change buffer defers non-unique secondary index page reads.

### Q5: Next-key lock range for `id=10 FOR UPDATE` on PK?

**Answer:** **Record lock only** on hit; inserting nearby ids is not blocked. Ranges/non-unique get next-key/gap.
**Follow-up:** Why can unique equality drop the gap? Uniqueness already forbids a duplicate in the gap.

### Q6: How does RR fight phantoms? Can MVCC alone?

**Answer:** Snapshot path: MVCC hides inserts. Current-read path: next-key. Snapshot then UPDATE can still see phantoms.

### Q7: GTID vs file/pos? Skip a bad event?

**Answer:** Auto-position and topology flexibility. Under GTID, inject empty tx with that GTID; do not blindly skip counters.

### Q8: Online DDL choice? PITR?

**Answer:** INSTANT → INPLACE → tools (gh-ost/pt-osc). PITR = physical/logical full + binlog to a point; rehearse restores.

### Q9: Deadlocks / deep pagination / covering index?

**Answer:** Conflicting lock order + wide locks from missing indexes. Prefer keyset over huge OFFSET. Covering avoids PK lookups.

### Q10: When to shard?

**Answer:** After schema, indexes, cache, and vertical scale fail. Complexity is the real cost.
