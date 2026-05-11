# Experiment: Disabling Primary Key Pruning in ClickHouse MergeTree

## 1. Objective
The goal of this experiment is to empirically demonstrate the critical role of primary key (PK) index pruning in the ClickHouse MergeTree storage engine. By intentionally disabling the logic that utilizes primary keys to skip data granules, we can observe the resulting performance degradation and I/O amplification when executing a highly selective query.

## 2. Methodology

### 2.1. Source Code Modification
To disable primary key pruning, we modified the core logic responsible for determining whether the primary key index can be used to filter data parts. This logic resides in the `MergeTreeDataSelectExecutor.cpp` file.

**File Path:** `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp`
**Function:** `MergeTreeDataSelectExecutor::markRangesFromPKRange`

**Original Logic:**
```cpp
bool key_condition_useful = !key_condition.alwaysUnknownOrTrue();
```

**Modified Logic:**
```cpp
// Experiment : Disable Primary Key pruning
bool key_condition_useful = false;
```

By hardcoding `key_condition_useful` to `false`, the query execution engine is forced to treat all primary key conditions as unhelpful. Consequently, the engine abandons index-based granule skipping (mark ranges filtering) and falls back to a full scan of the data part.

## 3. Experimental Setup and Execution

After compiling the modified ClickHouse binary, we ran a point-lookup query against a test table.

- **Target Table:** `pruning_exp.pk_test`
- **Query:** `SELECT * FROM pruning_exp.pk_test WHERE id = 4242424;`

This query is highly selective—it searches for exactly one specific `id` out of a large dataset. Under normal circumstances, the primary key index would allow ClickHouse to read only the few granules containing this `id`.

## 4. Results & Observations

We analyzed the execution metrics by querying the `system.query_log` for the specific query event.

**Extracted Metrics (from `system.query_log`):**
- **Query Duration (`query_duration_ms`):** 65 ms
- **Read Rows (`read_rows`):** 5,000,000 (5.00 million)
- **Read Bytes (`read_bytes`):** 40,087,136 (40.09 MB)
- **Result Rows (`result_rows`):** 1

**Analysis:**
Despite the query returning only **1 row**, the engine read **5 million rows** (over 40 MB of data). This indicates a complete full table scan. Because primary key pruning was disabled in the source code, ClickHouse had to read every single granule in the dataset, decompress the blocks, and evaluate the `WHERE id = 4242424` condition against every row.

## 5. Conclusion
This experiment clearly illustrates the fundamental importance of primary key indexing and pruning in the MergeTree engine. 
- **I/O Efficiency:** Primary keys allow ClickHouse to drastically minimize disk I/O. Without it, queries suffer from massive I/O amplification (reading 5 million rows to find 1).
- **CPU Overhead:** Full scans require significantly more CPU cycles to decompress blocks and filter rows that could have otherwise been skipped entirely using the sparse index.

Disabling the pruning mechanism immediately transforms an efficient $O(\log N)$ or $O(1)$ granule lookup into a brute-force $O(N)$ full table scan, highlighting why proper sorting keys are essential for performance in ClickHouse.
