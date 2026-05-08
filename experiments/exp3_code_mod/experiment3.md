# Experiment 3: Data Skew — Breaking the ORDER BY Assumption

> **Dataset:** 10 million rows with a single repeated value (`value = 1`)  
> **Table Under Test:** `exp_skew`  
> **Core Mechanism:** `ORDER BY value` with extreme skew — all rows share the same key value

---

## Hypothesis

ClickHouse MergeTree builds a **sparse primary index** over the `ORDER BY` column. The index is only useful when it can eliminate (prune) large ranges of granules. If data is **skewed** — where all or most rows share the same key — the index can locate the range but cannot skip any of it. Every granule must be read.

**Prediction:**
- Under uniform distribution: the index prunes aggressively → reads only a fraction of the data.
- Under extreme skew (`value = 1` for all rows): the index resolves the condition but the entire table matches → **full table scan**, no pruning benefit.
- `read_rows` ≈ 10 million, `result_rows` ≈ 10 million — no reduction despite an index existing.

---

## Step 1 — Create the Table (ORDER BY value)

```sql
CREATE TABLE exp_skew
(
    id    UInt32,
    value UInt32
)
ENGINE = MergeTree
ORDER BY value;
```

> **Critical design choice:** The `ORDER BY` column is `value`, not `id`. We are deliberately ordering by the column we will make degenerate.

**Observed output:**
```
Ok.

0 rows in set. Elapsed: 0.021 sec.
```

---

## Step 2 — Insert Skewed Data (Extreme Skew)

```sql
INSERT INTO exp_skew
SELECT
    number,
    1   -- ALL VALUES SAME (EXTREME SKEW)
FROM numbers(10000000);
```

Every row carries `value = 1`. The entire 10-million-row dataset is effectively at a **single point** in the sorted key space. The primary index has entries, but all of them point to granules containing `value = 1`.

**Observed output:**
```
10000000 rows in set. Elapsed: 0.868 sec. Processed 10.00 million rows, 80.00 MB
(11.52 million rows/s., 92.17 MB/s.)
Peak memory usage: 24.80 MiB.
```

---

## Step 3 — Run the Query

```sql
SELECT * FROM exp_skew WHERE value = 1;
```

**Observed output:**
```
10000000 rows in set. Elapsed: 0.418 sec. Processed 10.00 million rows, 80.00 MB
(23.93 million rows/s., 191.44 MB/s.)
Peak memory usage: 12.85 MiB.
```

The query returned all 10 million rows. Despite the primary index existing and `value` being the `ORDER BY` column, **zero pruning occurred** — every granule was read.

---

## Step 4 — Check Query Cost from system.query_log

```sql
SELECT
    read_rows,
    result_rows
FROM system.query_log
ORDER BY event_time DESC
LIMIT 1;
```

**Observed output:**
```
   ┌─read_rows─┬─result_rows─┐
1. │  10000000 │    10000000 │ -- 10.00 million
   └───────────┴─────────────┘

1 row in set. Elapsed: 0.010 sec. Processed 1.31 thousand rows, 15.78 KB
(129.63 thousand rows/s., 1.56 MB/s.)
Peak memory usage: 202.13 KiB.
```

---

## Step 5 — Results Summary

| Metric | Expected (uniform data) | Observed (skewed data) |
|---|---|---|
| `read_rows` | Small fraction of 10M | **10,000,000 (full scan)** |
| `result_rows` | 10M (all match `value = 1`) | **10,000,000** |
| Index pruning | High — most granules skipped | **None — all granules read** |
| Elapsed time | Fast (sub-100ms) | **0.418 sec** |
| Peak memory | Low | **12.85 MiB** |

---

## Step 6 — Why the Index Fails Under Skew

### What the primary index normally does

MergeTree's sparse index stores **one entry per `index_granularity` rows** (default: 8192). Each entry records the minimum `ORDER BY` key value for that granule. On a query like `WHERE value = 1`, the engine does a binary search across these index entries to find the first and last granules that could possibly contain `value = 1`.

**With uniform data**, `value` spans a wide range (e.g., 1 to 10,000,000). A query for `value = 1` resolves to a tiny range at the start of the sorted file — the engine skips the remaining 99%+ of granules.

**With skewed data (`value = 1` for all rows):**

```
Index entry 0:    min_value = 1    → could contain value=1  ✓
Index entry 1:    min_value = 1    → could contain value=1  ✓
Index entry 2:    min_value = 1    → could contain value=1  ✓
...
Index entry N:    min_value = 1    → could contain value=1  ✓
```

Every granule is a candidate. Binary search returns range `[0, N]` — the full file. **The index exists but cannot eliminate anything.**

### Internal execution path

```
1. InterpreterSelectQuery              [InterpreterSelectQuery.cpp]
   - Parses WHERE value = 1
   - Builds KeyCondition for the primary key (value)

2. MergeTreeDataSelectExecutor         [MergeTreeDataSelectExecutor.cpp]
   - Calls markRangesFromPKRange(value = 1)
   - Binary-searches sparse index:
       first mark where value could be ≥ 1  → mark 0
       last  mark where value could be ≤ 1  → mark N (end of file)
   - All marks included → mark_ranges = [0, total_marks]

3. MergeTreeRangeReader                [MergeTreeRangeReader.cpp]
   - Reads ALL granules (0 to N) sequentially
   - No granule is pruned

4. MergeTreeReader                     [MergeTreeReader.cpp]
   - Decompresses .bin data for every granule
   - All 10M rows delivered to the executor

5.  WHERE filter applied at row level
   - value = 1 → all 10M rows pass
   - result_rows = 10M
```

The index did its job correctly — it just could not help because **the key range is degenerate**. The issue is in the data distribution, not a code bug.

---

## Step 7 — Part B: Code-Level Analysis (Forced Full Scan)

### What the modification does

The SQL skew experiment demonstrates a realistic data problem. To prove the dependency on index pruning more mechanically, we can examine what would happen if pruning logic were disabled at the source level.

In `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp`, the function `markRangesFromPKRange()` computes the set of mark ranges to read. The result is stored in a `MarkRanges` object. If this result were replaced unconditionally with the full range:

```cpp
// Disable pruning — force full scan regardless of index
mark_ranges.clear();
mark_ranges.emplace_back(0, part->marks_count);
```

Then **every query, regardless of data distribution**, would read the entire table. This is the code-level analog of what extreme skew causes at the data level.

### What this confirms

| Scenario | Mechanism | Effect |
|---|---|---|
| Normal data, normal code | Index prunes mark ranges | Fast, selective reads |
| Skewed data (`value = 1`), normal code | Index cannot eliminate any mark | Full scan — data problem |
| Normal data, pruning disabled in code | Code ignores index output | Full scan — code problem |

Both the SQL skew experiment and the hypothetical code modification produce the **same observable outcome**: `read_rows = full table`. The difference is in the root cause:
- **SQL skew** → the index is structurally useless for this distribution.
- **Code modification** → the index is ignored by the executor.

This equivalence is exactly the point: MergeTree's read performance is **entirely dependent on the pruning pipeline working correctly with well-distributed data**.

---

## Step 8 — Internal Source Code Reference

| Component | Source File | Role in This Experiment |
|---|---|---|
| **Sparse index construction** | `src/Storages/MergeTree/MergeTreeIndexGranularity.cpp` | Builds the per-granule index structure at insert time |
| **Mark range resolution** | `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp` | `markRangesFromPKRange()` — binary-searches the index to find qualifying granule ranges |
| **KeyCondition** | `src/Storages/MergeTree/KeyCondition.cpp` | Translates the SQL `WHERE` predicate into a key-space interval for range searching |
| **Granule reading** | `src/Storages/MergeTree/MergeTreeRangeReader.cpp` | Reads the granules identified by `markRangesFromPKRange()` |
| **Column decompression** | `src/Storages/MergeTree/MergeTreeReader.cpp` | Decompresses `.bin` files for each granule |

Under extreme skew, `markRangesFromPKRange()` correctly computes the answer: all marks are in range. The problem is upstream — the data's lack of variance makes the index structurally unable to prune.

---

## Conclusion

| Question | Answer |
|---|---|
| Does a primary index on a skewed column help? | **No.** When all rows share the same key, every granule is a candidate — 0% pruning. |
| Does `read_rows = result_rows` always mean the index worked? | **No.** Here both are 10M because the entire table matches the predicate — the index still failed to prune anything. |
| What is the real assumption MergeTree makes? | That the `ORDER BY` column has **sufficient cardinality and distribution** to split the key space meaningfully. |
| What happens when that assumption breaks? | The engine degrades to a full table scan with zero index benefit, regardless of how many rows could theoretically be skipped. |



## References

| Source File | Role in This Experiment |
|---|---|
| `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp` | `markRangesFromPKRange()` — mark range resolution from primary index |
| `src/Storages/MergeTree/KeyCondition.cpp` | Translates `WHERE value = 1` into a primary key interval |
| `src/Storages/MergeTree/MergeTreeIndexGranularity.cpp` | Granularity structure — controls index density |
| `src/Storages/MergeTree/MergeTreeRangeReader.cpp` | Reads granules selected by the mark range resolver |
| `src/Storages/MergeTree/MergeTreeReader.cpp` | Physical column decompression per granule |
