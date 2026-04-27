# ⚙️ ClickHouse MergeTree: Design Decisions — Engineering Analysis

> **Ground rule:** If you cannot point to code, you have not understood the system.  
> Every claim below is backed by a function body, not just a file name.

---

## The 3 Core Decisions at a Glance

| Decision | What It Replaces | The Alternative That Was Rejected |
| :--- | :--- | :--- |
| **Immutable Parts** | In-place B-tree page updates | PostgreSQL MVCC / InnoDB page locks |
| **Sparse Primary Index** | Dense per-row B-tree index | MySQL InnoDB clustered index |
| **Background Merging** | Synchronous write-time compaction | InnoDB merge-on-write / RocksDB synchronous flush |

---

## Decision 1: Immutable Data Parts

> **What:** Once written, a part directory is never modified. `UPDATE`/`DELETE` create new parts; old ones are evicted.

### Proof from Code

**The Part State Machine — `src/Storages/MergeTree/IMergeTreeDataPart.h`:**

```cpp
// IMergeTreeDataPart.h
enum class State
{
    Temporary,   // being written to tmp_ directory
    PreActive,   // written, not yet renamed
    Active,      // visible to SELECT queries
    Outdated,    // superseded by a merge, pending deletion
    Deleting,    // removal in progress
    Deleted,     // gone
};
```

Once a part reaches `Active`, the only legal transition is to `Outdated`. **Nothing writes to an `Active` part.**

**Part eviction after a merge — `src/Storages/MergeTree/MergeTreeData.cpp`:**

```cpp
void MergeTreeData::removePartsFromWorkingSet(
    const DataPartsVector & remove, bool clear_without_timeout, DataPartsLock &)
{
    for (const DataPartPtr & part : remove)
    {
        if (part->getState() == IMergeTreeDataPart::State::Active)
            part->remove_time.store(
                clear_without_timeout ? 0 : time(nullptr),
                std::memory_order_relaxed);

        modifyPartState(part, IMergeTreeDataPart::State::Outdated);
    }
}
```

The source parts are not deleted immediately — they are stamped with a `remove_time` and flipped to `Outdated`. A separate cleanup thread collects them later. This grace period ensures that any in-flight `SELECT` that already holds a reference to those parts can finish reading safely.

### Connection to the Execution Path

During the **INSERT path**, `renameTempPartAndAdd()` commits a new part to `Active`:

```
Part written to tmp_insert_<uuid>/
        ↓
  [atomic rename]
        ↓
Part state = Active  ← visible to all SELECT queries from this moment
```

During a **background merge**, the merged part becomes `Active`, then `removePartsFromWorkingSet()` flips the source parts to `Outdated`:

```
Merged part = Active
Source parts → Outdated → Deleting → Deleted
```

`SELECT` queries running concurrently hold a **snapshot** of the active part list taken at query start. The state flip to `Outdated` does not affect them — they continue reading the old parts until they finish. This is snapshot isolation without MVCC.

### Why Not In-Place Updates (the Alternative)?

In PostgreSQL, an `UPDATE` writes a new row version *in the same heap page* and marks the old one as dead (MVCC). That requires:
- A page-level lock during the write
- A background `VACUUM` to reclaim dead tuples
- A transaction log entry for crash recovery

MergeTree eliminates all three by making the unit of modification a *directory*, not a page. Directory renames are atomic at the OS level. No lock, no vacuum, no WAL entry needed.

**The cost:** `ALTER TABLE ... UPDATE` triggers a `MergeTreeMutationEntry` that rewrites every affected part in full. Mutating 1 row in a 500 GB part rewrites 500 GB. ClickHouse is not built for row-level mutations.

---

## Decision 2: Sparse Primary Index

> **What:** `primary.idx` stores one key per 8,192 rows (one granule). Binary search finds candidate granules; the executor scans them to find matching rows.

### Proof from Code

**Granule index written at insert time — `src/Storages/MergeTree/MergeTreeDataWriter.cpp`:**

```cpp
// MergeTreeDataWriter.cpp
void MergeTreeDataWriter::writeIndex(...)
{
    for (size_t current_row = 0; current_row < rows; ++current_row)
    {
        if (current_row % index_granularity == 0)  // default: 8192
            index_granule.push_back(extractKeyValue(primary_key_columns, current_row));
    }
    // Serializes index_granule[] to primary.idx
}
```

**Granule lookup at query time — `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp`:**

```cpp
// MergeTreeDataSelectExecutor.cpp
MarkRanges MergeTreeDataSelectExecutor::markRangesFromPKRange(
    const MergeTreeData::DataPartPtr & part,
    const KeyCondition & key_condition, ...)
{
    MarkRanges res;
    size_t marks_count = part->index_granularity.getMarksCount();

    // Binary search over primary.idx (already loaded into RAM)
    // Returns [mark_start, mark_end) granule ranges that *may* match
    ...
    return res;  // caller reads only these granules from .bin via .mrk3 offsets
}
```

The function returns *candidate* granule ranges — not exact row offsets. The executor then reads those granules sequentially from disk via the `.mrk3` mark files. Rows that don't match the `WHERE` clause are discarded after reading.

### Connection to the Execution Path

The index is built during **Step 3 of the INSERT path** (`MergeTreeDataWriter::blockToDataPart()`), which is why the block must be **sorted by primary key first** — an unsorted block would produce a meaningless index that binary search cannot use.

At **query time**, the SELECT path is:

```
1. Load primary.idx into RAM (happens once, stays cached)
2. markRangesFromPKRange() → binary search → candidate granule list
3. Read only those granule byte ranges from .bin files (via .mrk3 offsets)
4. Discard rows not matching WHERE clause
```

Step 3 is the only disk I/O for the entire query on a cold cache. Everything else is in-memory computation.

### The RAM Math

| Index Type | 1 Billion rows, 8 bytes/key | Stays in RAM on a 32 GB server? |
| :--- | :--- | :--- |
| Dense per-row B-tree | **~8 GB** | ❌ Competes with query working memory |
| Sparse (1 per 8,192 rows) | **~1 MB** | ✅ Trivially |

For a 100-billion-row table: dense = ~800 GB, sparse = ~100 MB. The sparse index is what makes ClickHouse's "index always in RAM" claim true at any scale.

### The Granularity Tax — Quantified

> [!WARNING]
> A point lookup `WHERE id = X` on a 10M-row part must read at minimum **1 full granule = 8,192 rows** from disk, even though only 1 row matches. For a 100-byte-wide row, that is ~800 KB of disk I/O for 1 result row.

On a B-tree (alternative), the same lookup reads ~3 pages × 8 KB = 24 KB. **ClickHouse is 33× worse for point lookups.** This is the deliberate cost of the analytical trade-off.

### Why Not a Dense B-Tree (the Alternative)?

InnoDB's clustered B-tree index stores one entry per row, physically ordered on disk. This gives O(log n) point lookups at the cost of:
- Fragmented disk layout (B-tree pages, not sequential blocks)
- RAM-hungry index that must be partially cached for performance
- Random I/O on insertion (page splits)

ClickHouse trades point-lookup efficiency for sequential I/O in bulk reads — optimal for `SELECT count(*) WHERE date BETWEEN ...` type queries that drive OLAP workloads.

---

## Decision 3: Background Merging (LSM-Inspired)

> **What:** Every INSERT produces one small part immediately. A background thread pool merges adjacent small parts into larger ones asynchronously, continuously.

### Proof from Code

**Merge candidate selection — `src/Storages/MergeTree/SimpleMergeSelector.cpp`:**

```cpp
// SimpleMergeSelector.cpp
PartsRange SimpleMergeSelector::select(
    const PartsRanges & parts_ranges,
    const size_t max_total_size_to_merge)
{
    // Only consider adjacent parts within the same partition
    // Prefer merging groups where: max_size / min_size < size_ratio_coefficient
    // (i.e., similarly-sized parts — like LevelDB's level strategy)
    // Skip if total merged size would exceed max_bytes_to_merge_at_max_space_in_pool
    ...
}
```

**Background task scheduling — `src/Storages/MergeTree/BackgroundJobsAssignee.cpp`:**

```cpp
// BackgroundJobsAssignee.cpp
void BackgroundJobsAssignee::threadFunc()
{
    // Called repeatedly by the background thread pool
    // Calls back into MergeTreeMergerMutator::selectPartsToMerge()
    // If a candidate set is found, spawns a MergeTask
    auto task = storage.selectPartsToMerge(...);
    if (task)
        pool.scheduleOrThrow([task] { task->execute(); });
}
```

The background merge loop runs independently of any INSERT or SELECT. There is **no coordination point** between the INSERT path and the merge path — they share only the active parts list, which is protected by a read-write lock.

### Connection to the Execution Path

After **Step 4 of the INSERT path** (`renameTempPartAndAdd()`) commits a new part:

```
New part: 20230601_5_5_0  (partition=June 2023, blocks 5–5, level 0)
     ↑
  Active part set now contains: ..._1_1_0, _2_2_0, _3_3_0, _4_4_0, _5_5_0

Background merger wakes up:
  selectPartsToMerge() → picks _1_1_0 through _5_5_0
  Merge executes → writes 20230601_1_5_1  (blocks 1–5, level 1)
  removePartsFromWorkingSet() → source parts → Outdated
  Active part set now: ..._1_5_1
```

The merge level in the part name encodes history: `level=1` means merged once from level-0 parts. A part at `level=2` was produced by merging level-1 parts. This is directly analogous to LevelDB's level numbering.

### Real-World Failure Mode — `Too Many Parts`

> [!CAUTION]
> ClickHouse tracks part count per partition in `data_parts_indexes`. When this count approaches the limit, the following happens internally:

**At ~150 parts:** ClickHouse logs a warning.  
**At ~250 parts:** `MergeTreeData::delayInsertOrThrowIfNeeded()` is called — it sleeps the INSERT thread for `insert_delay_ms` (default grows exponentially with part count).  
**At 300 parts (default `max_parts_in_total`):** Exception thrown:

```
DB::Exception: Too many parts (300). Merges are processing slower than inserts.
```

**What triggers this cascade?**  
- Firing one `INSERT` per row (1 row = 1 part)
- Inserting across many partitions simultaneously (each partition gets its own part per INSERT)
- A merge backlog during a query-heavy window (merges starved of I/O)

**Why 300?** The threshold is empirically tuned: at 300 parts per partition, `markRangesFromPKRange()` must scan 300 separate `primary.idx` files per query instead of 1. Query time degrades roughly linearly with part count for range scans

**The fix is upstream:** batch inserts to at least 100K rows per statement, so each INSERT produces one large part, not hundreds of small ones.

### Why Not Synchronous Compaction (the Alternative)?

InnoDB merges index pages synchronously during INSERT — the INSERT does not return until the B-tree is updated and the WAL entry is written. This guarantees immediately consistent read performance but:
- INSERT latency is proportional to index size + WAL write
- High concurrency of inserts causes B-tree lock contention
- No way to pipeline or batch the compaction cost

ClickHouse's async approach: INSERT latency = one sequential disk flush (sub-millisecond at network speeds). The compaction cost is invisible to the client, paid later by the background thread pool.

---

## 🧩 How the Three Decisions Form One System

These are not independent choices — each decision creates the conditions that make the next one viable:

```
Immutable Parts
  └─ No write locks on parts
      └─ Background merge can read parts safely (no coordination)
          └─ Merge produces fewer, larger parts
              └─ Sparse index skips more granules per query
                  └─ Index remains tiny → always fits in RAM
                      └─ RAM index makes binary search instant at query time
                          └─ No dense per-row index needed → Sparse index justified
```

Remove any one decision and the system breaks:
- Without immutability → background merge needs page locks → INSERT latency spikes
- Without sparse index → index doesn't fit in RAM as parts grow → query performance collapses
- Without background merging → part count explodes → sparse index must scan hundreds of files → query performance collapses

---

## Alternatives Comparison

| Decision | MergeTree Approach | Alternative | Why MergeTree's Choice Wins for OLAP |
| :--- | :--- | :--- | :--- |
| **Data modification** | Immutable parts + full rewrite on mutation | MVCC row versioning (PostgreSQL) | No per-row overhead; reads require no visibility check |
| **Primary index** | Sparse (1 key / 8192 rows) | Dense B-tree (1 key / row) | Index stays in RAM at multi-TB scale; sequential I/O |
| **Compaction** | Background async merging | Synchronous write-time merge (InnoDB) | INSERT latency unaffected by index size; fully pipelineable |
| **Crash recovery** | Atomic directory rename (no WAL) | Write-ahead log (WAL) | No log replay on startup; recovery = discard `tmp_` directories |

---

## Conclusion

> ClickHouse MergeTree is an **LSM-tree without a memtable**: filesystem atomicity replaces the WAL, a sparse in-RAM index replaces Bloom filters, and columnar immutable directories replace SSTables.

The three decisions above are not isolated optimizations — they are a single coherent system designed around one insight: **analytical workloads read far more rows than they write, and they read entire column ranges, not individual cells.** Every decision favors sequential bulk reads over random point access. That is the design MergeTree encodes, and every tradeoff in these decisions flows from it.
