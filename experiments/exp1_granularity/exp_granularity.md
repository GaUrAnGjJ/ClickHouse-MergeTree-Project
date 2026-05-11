# Experiment 1: Impact of Index Granularity on Query Performance

## 1. Objective
The goal of this experiment is to evaluate the impact of the `index_granularity` setting on ClickHouse's read performance and I/O efficiency. We investigate how changing the default index granularity from 8192 to 128 rows affects the number of rows read from disk during a highly selective point query.

## 2. Methodology

### 2.1. Source Code Modification
ClickHouse defines its default storage engine settings in its C++ source code. We modified the default `index_granularity` for MergeTree tables by editing the ClickHouse source file.

**File Path:** `src/Storages/MergeTree/MergeTreeSettings.cpp`

**Original Logic (Default: 8192):**
```cpp
DECLARE(UInt64, index_granularity, 8192, R"(
Maximum number of data rows between the marks of an index. I.e how many
correspond to one primary key value.
)", 0) \
```

**Modified Logic (Custom: 128):**
```cpp
// Experiment : 1 Change Index Granulairty from 8192 to 128 .
DECLARE(UInt64, index_granularity, 128, R"(
Maximum number of data rows between the marks of an index. I.e how many
correspond to one primary key value.
)", 0) \
```

By altering this setting in the C++ source and rebuilding ClickHouse, the global default index granularity drops to 128, creating much smaller data blocks (granules) between sparse index marks.

## 3. Experimental Setup and Execution

We created two distinct tables to compare the granularities:
1. `project.test_coarse`: Created using the standard **8192** granularity.
2. `project.test_fine`: Created using the modified **128** granularity.

We then executed an identical point-lookup query on both tables:
```sql
SELECT * FROM table WHERE id = 777777;
```
Because this query matches exactly 1 row, it is the perfect candidate to reveal the size of the underlying data granules being pulled from disk.

## 4. Results & Observations

We extracted the query execution metrics from the `system.query_log` table.

**Query Executed to Fetch Metrics:**
```sql
SELECT
    query_duration_ms,
    read_rows,
    read_bytes,
    result_rows,
    query
FROM system.query_log
WHERE (type = 'QueryFinish') AND (query LIKE '%777777%')
ORDER BY event_time DESC
LIMIT 4;
```

### 4.1. Execution Metrics

**Result 1: Coarse Granularity (`test_coarse`, Granularity = 8192)**
- **Query:** `SELECT * FROM project.test_coarse WHERE id = 777777;`
- **Read Rows:** 8,192
- **Read Bytes:** 188,416 (~188 KB)
- **Result Rows:** 1
- **Duration:** 16 ms

**Result 2: Fine Granularity (`test_fine`, Granularity = 128)**
- **Query:** `SELECT * FROM project.test_fine WHERE id = 777777;`
- **Read Rows:** 128
- **Read Bytes:** 1,724 (~1.7 KB)
- **Result Rows:** 1
- **Duration:** 30 ms

### 4.2. Analysis
- **I/O Overhead:** Because ClickHouse utilizes a sparse index, it cannot locate a single row perfectly. It must locate the correct mark (granule start) and read the entire granule. 
- For `test_coarse`, finding the single matching record required reading **all 8,192 rows** in the granule from disk.
- For `test_fine`, the engine only had to read **128 rows** to locate the matching record.
- **Data Read Reduction:** The fine-grained table resulted in a **~99% reduction in bytes read** (188 KB vs 1.7 KB) and processed exactly 64 times fewer rows ($8192 / 128 = 64$).
- **Duration Consideration:** The data read volume was significantly lower for `test_fine`, yet the query duration was slightly higher (30 ms vs 16 ms). This occurs because finer granularity drastically increases the number of marks and the physical size of the primary `.idx` file. As a result, the initial binary search phase of the index evaluation carries slightly more computational overhead.

## 5. Conclusion
This experiment provides empirical evidence of how MergeTree's sparse indexing dictates physical read behavior. The `index_granularity` setting defines the smallest indivisible chunk of data ClickHouse will read. 
- A **larger granularity (8192)** minimizes RAM usage and CPU overhead for the index search, making it ideal for large analytical range scans, but suffers from I/O amplification during highly selective point lookups.
- A **smaller granularity (128)** almost eliminates I/O amplification for point queries, but at the cost of a larger index footprint and slightly slower index resolution.
