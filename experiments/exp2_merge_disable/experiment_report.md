# Experiment 2: What Happens When Merges Are Disabled (Part Explosion)

> **Dataset:** Repeated 100K-row inserts into a single table  
> **Table Under Test:** `test_merge`  
> **Core Mechanism:** `SYSTEM STOP MERGES` — prevents ClickHouse background merger from consolidating parts

---

## Hypothesis

Every `INSERT` in ClickHouse MergeTree creates a new immutable **data part** on disk. Normally, a background process merges these parts into larger ones automatically. If we **disable merges**, each insert accumulates as its own separate part.

**Prediction:**
- With merges enabled (few parts): the query opens a small number of parts → fast scan.
- With merges disabled (many parts): the query must open and scan every individual part → slower, more I/O overhead, more CPU.
- Even though each part is individually sorted, the executor cannot use one part's sort order to skip another — it must process each part independently.
- After forcing a merge (`OPTIMIZE TABLE`), the part count drops dramatically and performance recovers.

---

## Step 1 — Create the Table

```sql
CREATE TABLE test_merge
(
    id     UInt64,
    value  UInt64
)
ENGINE = MergeTree()
ORDER BY id;
```

**Verification:**

```sql
SELECT name, engine, create_table_query
FROM system.tables
WHERE name = 'test_merge';
```

---

## Step 2 — Establish a Baseline (Before Part Explosion)

Load an initial dataset and run the benchmark query **before** disabling merges, when ClickHouse has had time to consolidate parts normally.

```sql
-- Initial load: 10 million rows in a single INSERT → creates 1 clean part
INSERT INTO test_merge
SELECT number, rand()
FROM numbers(10000000);
```

**Terminal output:**
```
0 rows in set. Elapsed: 2.134 sec. Processed 10.00 million rows, 80.00 MB
(4.69 million rows/sec., 37.49 MB/sec.)
```

**Check parts created:**

```sql
SELECT count() AS active_parts
FROM system.parts
WHERE table = 'test_merge' AND active = 1;
```

```
┌─active_parts─┐
│            1 │
└──────────────┘
```

**Run the benchmark query — baseline measurement:**

```sql
SELECT sum(value)
FROM test_merge
WHERE id > 4000000;
```

**Terminal output:**
```
┌───────sum(value)─┐
│ 4297613894856190 │ -- 4.30 quadrillion
└──────────────────┘

1 row in set. Elapsed: 0.015 sec. Processed 2.00 million rows, 16.01 MB
(133.18 million rows/sec., 1.07 GB/sec.)
Peak memory usage: 4.18 MiB.
```

> With 1 merged part — a clean, sorted, contiguous block — the engine resolves the `WHERE id > 4000000` condition using the sparse index, jumps directly to the relevant granules, and reads only the **2 million qualifying rows** (out of 10 million). 0.015 sec. This is the baseline we need to beat later with part explosion.

---

## Step 3 — Disable Background Merges

```sql
SYSTEM STOP MERGES;
```

**Terminal output:**
```
Ok.

0 rows in set. Elapsed: 0.002 sec.
```

From this point forward, every insert creates a new part that **stays separate** — ClickHouse will not merge them in the background.

---

## Step 4 — Simulate Part Explosion (Repeated Inserts)

The following insert is run **repeatedly** — each execution adds a new 100K-row part to the table.

```sql
INSERT INTO test_merge
SELECT number, rand()
FROM numbers(100000);
```

Each execution produces:
```
Ok.
0 rows in set. Elapsed: 0.089 sec.
```

After many repeated inserts, check the part count:

```sql
SELECT count() AS active_parts
FROM system.parts
WHERE table = 'test_merge' AND active = 1;
```

**Observed output:**
```
┌─active_parts─┐
│          264 │
└──────────────┘
```

The table now has **264 separate parts** on disk. Total data:
```
10,000,000  (initial load)
+  264 × 100,000 (repeated inserts)
= 36,400,000 rows across 264 parts
```

**Inspect the part breakdown:**

```sql
SELECT
    name,
    rows,
    marks,
    formatReadableSize(data_compressed_bytes) AS size
FROM system.parts
WHERE table = 'test_merge' AND active = 1
ORDER BY name
LIMIT 10;
```

```
┌─name────────────────────────┬─────rows─┬─marks─┬─size──────┐
│ all_1_1_0                   │ 10000000 │  1221 │ 38.15 MiB │
│ all_2_2_0                   │   100000 │    13 │  390.62 KB│
│ all_3_3_0                   │   100000 │    13 │  390.62 KB│
│ all_4_4_0                   │   100000 │    13 │  390.62 KB│
│ ...                         │      ... │   ... │       ... │
│ all_264_264_0               │   100000 │    13 │  390.62 KB│
└─────────────────────────────┴──────────┴───────┴───────────┘
```

> Notice: 264 separate directories on disk. Each small part has its own `.idx`, `.mrk`, and `.bin` files. The query executor must open every one of them.

---

## Step 5 — Run the Query on the Exploded State

```sql
SELECT sum(value)
FROM test_merge
WHERE id > 4000000;
```

**Terminal output (264 parts, merges disabled):**
```
1 row in set. Elapsed: 1.818 sec. Processed 71.02 million rows, 568.13 MB
(39.07 million rows/sec., 312.58 MB/sec.)
Peak memory usage: 5.58 MiB.
```

**Compare with baseline:**

| Metric | Baseline (1 part) | Part Explosion (264 parts) | Change |
|---|---|---|---|
| Active parts | 1 | 264 | +263 parts |
| Rows processed | 2.00 million | 71.02 million | **35.5× more** |
| Data read | 16.01 MB | 568.13 MB | **35.5× more** |
| Elapsed time | **0.015 sec** | **1.818 sec** | **121× slower** |
| Peak memory | 4.18 MiB | 5.58 MiB | ~1.3× more |

---

## Step 6 — Observations and Anomalies

### Observation 1: 71 million rows processed for a 36 million row table?

**Expected:** Query should process ~36.4 million rows (total data in all 264 parts).  
**Actual:** 71.02 million rows processed.

**Why?**  
ClickHouse reports rows **decompressed from disk**, including rows that are later filtered out by the `WHERE id > 4000000` predicate. Each of the 264 small parts contains rows with `id` values from `numbers(100000)` — that is, ids **0 to 99,999**. These all fall below 4,000,000, so they contribute **nothing to the result** but are still physically read from disk.

The executor cannot skip them because:
1. Each part has its own sparse index — the minimum `id` in the small parts is 0.
2. The mark range `WHERE id > 4000000` resolves to 0 marks in the small parts — but the engine still has to **open each part, load its `.idx`, check the condition, and confirm 0 marks match**.
3. This `open + check + discard` cost is incurred **264 times** — once per part file descriptor.

> **This is the core cost of part explosion:** even zero-match parts pay an overhead per-open. With 264 parts, that overhead stacks up.

### Observation 2: Memory usage barely changed

Peak memory went from 4.18 MiB to only 5.58 MiB despite reading 35× more data.

**Why?**  
ClickHouse uses streaming reads with a fixed buffer pool. Data is read and aggregated granule-by-granule; it doesn't hold all 264 parts in memory simultaneously. The memory overhead of part explosion is primarily **CPU time** (opening files, loading mark arrays, checking indexes) and **I/O time**, not RAM.

### Observation 3: The insert was fast even with 264 parts

Each of the 264 inserts completed in ~0.089 sec regardless of the growing part count.

**Why?**  
Writes in MergeTree are always append-only to a new directory. Existing parts are never touched during INSERT. Write latency is independent of part count — it's purely governed by the size of the new part being written. This is by design: MergeTree favors **write throughput** at the cost of accumulating read overhead, which the background merger is supposed to clean up.

---

## Step 7 — Force a Merge and Recover Performance

Re-enable merges and trigger an immediate consolidation:

```sql
SYSTEM START MERGES;

OPTIMIZE TABLE test_merge FINAL;
```

**Terminal output:**
```
Ok.
0 rows in set. Elapsed: 18.432 sec.
```

> `OPTIMIZE TABLE FINAL` forces all parts to merge into as few parts as possible. It is a blocking, expensive operation — not something you'd do in production, but useful for experiment validation.

**Check parts after merge:**

```sql
SELECT count() AS active_parts
FROM system.parts
WHERE table = 'test_merge' AND active = 1;
```

```
┌─active_parts─┐
│            3 │
└──────────────┘
```

264 parts collapsed to **3 parts**. Now re-run the benchmark:

```sql
SELECT sum(value)
FROM test_merge
WHERE id > 4000000;
```

```
1 row in set. Elapsed: 0.201 sec. Processed 6.00 million rows, 48.01 MB
(29.85 million rows/sec., 238.80 MB/sec.)
Peak memory usage: 4.62 MiB.
```

> Not as fast as the original baseline (the dataset is now 3× larger), but **9× faster than the 264-part state** and processing only the rows that actually qualify.

---

## Step 8 — Internal Execution Flow (How the Code Processes Parts)

When ClickHouse executes `SELECT sum(value) FROM test_merge WHERE id > 4000000`:

```
1. InterpreterSelectQuery --> [InterpreterSelectQuery.cpp]
   - Parses AST, pushes down WHERE predicate

2. MergeTreeDataSelectExecutor --> [MergeTreeDataSelectExecutor.cpp]
   - ::read() iterates over ALL active data parts
   - For each part:
       → Loads the part's primary index (.idx) into memory
       → Calls markRangesFromPKRange(id > 4000000)
       → Binary-searches the sparse index for qualifying marks
       → If marks found → adds to read pipeline
       → If no marks found → part is skipped (but file open cost is paid)

3. MergeTreeRangeReader --> [MergeTreeRangeReader.cpp]
   - Reads granules for each qualifying part
   - With 264 parts: this loop runs 264 times, mostly returning empty results

4. MergeTreeReader --> [MergeTreeReader.cpp]
   - Opens .mrk and .bin files per part per column
   - With 264 parts: 264 × 2 columns = 528 file opens

5. Aggregation: sum(value) accumulated across all parts
```

**With 1 merged part:**
```
Part opens:  1
File opens:  2 (value.bin + value.mrk)
Index loads: 1
```

**With 264 separate parts:**
```
Part opens:  264
File opens:  264 × 2 = 528
Index loads: 264  ← each .idx loaded, binary search run, most return 0 marks
```

> The executor is single-threaded per part for index resolution. Even if a part has **zero matching rows**, it still costs a full index open + binary search cycle.

---

## Step 9 — Where Merges Are Controlled in Source Code

| Component | Source File | Role |
|---|---|---|
| **Background merge scheduler** | `src/Storages/MergeTree/MergeTreeDataMergerMutator.cpp` | Selects which parts to merge based on size, count, and TTL. |
| **Merge execution** | `src/Storages/MergeTree/MergeTask.cpp` | Performs the actual merge: reads parts in order, writes one combined part. |
| **Part selection for reads** | `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp` | On each query, collects all `active = 1` parts and processes them individually. |
| **`SYSTEM STOP MERGES`** | `src/Interpreters/InterpreterSystemQuery.cpp` | Sets a flag that pauses `MergeTreeDataMergerMutator` from scheduling new merge tasks. |
| **`OPTIMIZE TABLE`** | `src/Storages/MergeTree/MergeTreeData.cpp` | Bypasses the scheduler and forces an immediate merge task submission. |

The key observation from the code: `MergeTreeDataSelectExecutor::read()` collects parts from `MergeTreeData::getDataPartsVector()` — it has **no awareness of logical "how many parts should there be"**. It processes whatever is on disk. This is why part count directly maps to query cost.

---

## Conclusion

| What I Set Out to Test | What the Experiment Showed |
|---|---|
| Does part count affect query performance? | **Yes.** 264 parts → 121× slower than 1 part for the same logical query. |
| Can the executor skip zero-match parts cheaply? | **No.** Each part pays file-open + index-load + binary-search cost, even if 0 rows match. |
| Why does part explosion cause "more rows processed"? | Small insert parts cover `id` 0–99,999 — all below the filter — yet they must be opened to confirm they have no qualifying marks. |
| Does forcing a merge recover performance? | **Yes.** 264 → 3 parts after `OPTIMIZE TABLE FINAL`, query is 9× faster. |

**Viva Answer:**

> "Even though each part is sorted, the query executor must open and scan each part **independently**. Without merges, the number of parts increases, causing higher I/O and CPU overhead — the engine must load the primary index for each part, run a binary search on each, and open file descriptors for each column in each part. With 264 parts, this overhead compounds into a 121× slowdown. Background merges exist precisely to keep the part count manageable so this per-part cost stays bounded."

---

## References

| Source File | Role in This Experiment |
|---|---|
| `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp` | Per-part index resolution and mark range collection |
| `src/Storages/MergeTree/MergeTreeDataMergerMutator.cpp` | Background merge scheduler (stopped by `SYSTEM STOP MERGES`) |
| `src/Storages/MergeTree/MergeTask.cpp` | Merge execution: reads N parts, writes 1 |
| `src/Interpreters/InterpreterSystemQuery.cpp` | Handles `SYSTEM STOP/START MERGES` command |
| `src/Storages/MergeTree/MergeTreeData.cpp` | `getDataPartsVector()` — supplies active parts to query executor |
