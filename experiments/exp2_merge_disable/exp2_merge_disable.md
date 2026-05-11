# Experiment 2: Disabling Background Merges (MergeTree)

## 1. Objective
The goal of this experiment is to observe the behavior of ClickHouse's MergeTree storage engine when background merges are intentionally disabled via source code modification. We aim to demonstrate how multiple unmerged data parts accumulate on disk and how they can be forcefully consolidated using the `OPTIMIZE TABLE` command.

## 2. Methodology

### 2.1. Source Code Modification
To manipulate the background merge process, we modified the merge selection logic in the ClickHouse source code.

**File Path:** `src/Storages/MergeTree/MergeTreeDataMergerMutator.cpp`
**Function:** `MergeTreeDataMergerMutator::selectPartsToMerge`

**Original Logic:**
```cpp
auto ranges = splitByMergePredicate(std::move(collected.ranges), merge_predicate, series_log);
if (ranges.empty())
{
    return std::unexpected(SelectMergeFailure{
        .reason = SelectMergeFailure::Reason::CANNOT_SELECT,
        .explanation = PreformattedMessage::create("No parts satisfy preconditions for merge"),
    });
}
```

**Modified Logic:**
```cpp
auto ranges = splitByMergePredicate(std::move(collected.ranges), merge_predicate, series_log);
if (ranges.empty())
{
// Experiment 2 MergeTree Disable ( MergeTree )
    return std::unexpected(SelectMergeFailure{
        .reason = SelectMergeFailure::Reason::CANNOT_SELECT,
        .explanation = PreformattedMessage::create("Background merges disabled for experiment")
        //.explanation = PreformattedMessage::create("No parts satisfy preconditions for merge"),
    });
}
```
*Note: The explanation message was modified to trace the experiment and indicate that background merges are disabled.*

## 3. Experimental Execution

With the custom-built ClickHouse running, we utilized a test table named `merge_exp.parts_test`. Because of the disabled background merges, data inserts were written to separate data parts on disk and remained unconsolidated.

### 3.1. Checking Active Parts
First, we checked the `system.parts` table to observe how many active parts had accumulated.

**Query:**
```sql
SELECT
    table,
    count(*) AS active_parts
FROM system.parts
WHERE database = 'merge_exp'
    AND table = 'parts_test'
    AND active
GROUP BY table;
```

**Terminal Output:**
```text
Query id: 0352b34e-1d79-4386-baa4-c61710faa2c5

┌─table──────┬─active_parts─┐
│ parts_test │            5 │
└────────────┴──────────────┘

1 row in set. Elapsed: 0.039 sec.
```
As expected, the table had **5 active parts**. The background merger did not combine them automatically.

### 3.2. Forcing a Merge
We then bypassed the background merge scheduler and forced all active parts to merge into a single part using the `OPTIMIZE TABLE` command with the `FINAL` modifier.

**Query:**
```sql
OPTIMIZE TABLE merge_exp.parts_test FINAL;
```

**Terminal Output:**
```text
Query id: b9aa2892-16f8-47d2-8b34-781b78945ef3

Ok.

0 rows in set. Elapsed: 0.017 sec.
```

### 3.3. Verifying the Merge
To confirm the merge was successful, we re-ran the query checking the `system.parts` table.

**Query:**
```sql
SELECT
    table,
    count(*) AS active_parts
FROM system.parts
WHERE database = 'merge_exp'
    AND table = 'parts_test'
    AND active
GROUP BY table;
```

**Terminal Output:**
```text
Query id: e74b291a-386b-4066-9954-af7227e2cea8

┌─table──────┬─active_parts─┐
│ parts_test │            1 │
└────────────┴──────────────┘

1 row in set. Elapsed: 0.010 sec.
```

## 4. Conclusion
This experiment successfully demonstrates that:
1. Modifying the C++ logic inside `MergeTreeDataMergerMutator::selectPartsToMerge` or interfering with the merge background tasks leads to the accumulation of independent data parts (e.g., 5 unmerged parts).
2. The `OPTIMIZE TABLE ... FINAL` command provides a mechanism to manually trigger a merge. It effectively consolidates all separate active parts into a single, clean part on disk (reducing the count from 5 to 1), bypassing the standard background scheduling logic.
