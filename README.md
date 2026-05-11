# 🔍 Cracking Open ClickHouse's MergeTree Engine

> *We took a database engine that processes billions of rows per second, ripped open its source code (305 files), ran experiments that broke it on purpose, and documented everything we learned.*

This is not a tutorial. This is a **reverse-engineering journal** — a semester-long investigation into how ClickHouse's MergeTree storage engine actually works under the hood, told through the code we read, the experiments we ran, and the failures that taught us the most.

---

## The Story in 60 Seconds

It started with a simple question: *"How does ClickHouse scan 10 million rows faster than most databases scan 10 thousand?"*

The answer, we discovered, lives in three architectural bets that ClickHouse made — and that most databases didn't:

1. **Parts are immutable.** Once data hits disk, it's never modified. Updates? Rewrite the entire part. This sounds insane until you realize it eliminates every lock, every WAL entry, and every page-level conflict.

2. **The index is deliberately imprecise.** One index entry per 8,192 rows instead of one per row. The primary index for a billion-row table fits in ~1 MB of RAM. The trade-off? A point query must read at least 8,192 rows to find one.

3. **Merging happens in the background.** Every INSERT creates a new folder. A background crew quietly combines small folders into big ones. If that crew stops? We found out. Queries became **121× slower.**

We proved each of these by breaking the system on purpose.

---

## What We Actually Did

### 📖 Phase 1: Read the Source Code

We cloned 305 source files from ClickHouse's `MergeTree/` directory and traced two paths through the code:

- **The INSERT path** — from SQL parser → interpreter → sort block → write columns → atomic rename
- **The SELECT path** — from SQL parser → load sparse index → binary search → read granules → filter rows

Every function call, every file handoff, every design decision is documented with code snippets and explanations in plain English.

### 🧪 Phase 2: Run Experiments That Break Things

| Experiment | What We Did | What Broke |
|---|---|---|
| **[Exp 1: Granularity](experiments/exp1_granularity/experiment_report.md)** | Compared `index_granularity = 128` vs `8192` on 10M rows | The "precise" table read 42 rows but was **2.9× slower** than the "imprecise" one that read 8,192 rows |
| **[Exp 2: Part Explosion](experiments/exp2_merge_disable/experiment_report.md)** | Disabled background merges, created 264 separate parts | Queries became **121× slower**. The engine processed 71M rows from a 36M row table. |
| **[Exp 3: Data Skew](experiments/exp3_code_mod/experiment3.md)** | Inserted 10M rows where every single value was identical | The primary index existed, loaded, ran a binary search — and achieved absolutely nothing. Full table scan. |

### 🩻 Phase 3: Perform the Autopsy

Every surprise from the experiments was traced back to the source code. Every "bug" turned out to be a feature. We documented 10 failures, ranked them by severity, and mapped each one to the exact function in the codebase where it originates.

---

## Repository Map

```
ClickHouse-MergeTree-Project/
│
├── 📁 raw/MergeTree/                    # 305 source files from ClickHouse
│   ├── MergeTreeData.cpp                #   (475 KB — the biggest file)
│   ├── MergeTreeDataSelectExecutor.cpp  #   (the SELECT engine)
│   ├── MergeTreeDataWriter.cpp          #   (the INSERT engine)
│   ├── MutateTask.cpp                   #   (UPDATE/DELETE — rewrites entire parts)
│   ├── KeyCondition.cpp                 #   (216 KB — WHERE clause → index lookup)
│   └── ...301 more files
│
├── 📁 code-notes/                       # Our analysis documents
│   ├── execution_flow.md                #   INSERT path — 5 steps, code-traced
│   ├── design_decisions.md              #   Why immutability, sparse index, async merges
│   ├── concept_mapping.md               #   20 concepts → source files → plain English
│   └── failure_analysis.md              #   10 failures ranked, explained, and mapped to code
│
├── 📁 experiments/
│   ├── exp1_granularity/                #   Granularity 128 vs 8192 — the sniper vs shotgun test
│   │   └── experiment_report.md
│   ├── exp2_merge_disable/              #   264 parts — the part explosion experiment
│   │   └── experiment_report.md
│   └── exp3_code_mod/                   #   Data skew — the index that worked but achieved nothing
│       └── experiment3.md
│
├── 📁 diagrams/                         #   Architecture and flow diagrams
└── 📁 report/                           #   Final project report
```

---

## The Numbers That Surprised Us

```
┌─────────────────────────────────────────────────────────────────┐
│  42          Rows read by a "precise" index for 1 matching row  │
│              (we expected 128)                                   │
│                                                                  │
│  121×        Slowdown from 264 unmerged parts vs 1 merged part   │
│                                                                  │
│  71,000,000  Rows "processed" from a table with only 36,400,000 │
│                                                                  │
│  ~1 MB       Index size for 1 BILLION rows (sparse index magic)  │
│                                                                  │
│  0           Granules pruned when every row has the same value    │
│                                                                  │
│  200 GB      I/O cost of changing 1 row in a 100 GB part         │
│              (read old + write new)                               │
│                                                                  │
│  18.4 sec    Time to merge 264 parts back into 3                 │
│              (the cost of skipping background merges)             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Guide — Where to Start

**If you want to understand the architecture:**
→ Start with [`execution_flow.md`](code-notes/execution_flow.md) — traces an INSERT from SQL to disk in 5 steps

**If you want to understand *why* the architecture is the way it is:**
→ Read [`design_decisions.md`](code-notes/design_decisions.md) — compares MergeTree's choices against PostgreSQL, InnoDB, and RocksDB

**If you want a map of the codebase:**
→ Open [`concept_mapping.md`](code-notes/concept_mapping.md) — 20 concepts, each linked to source files with plain-English explanations

**If you want the most interesting part:**
→ Read [`failure_analysis.md`](code-notes/failure_analysis.md) — 10 failures, ranked by severity, each with an analogy, a code trail, and a lesson learned

**If you want to see the raw evidence:**
→ Jump into the [`experiments/`](experiments/) folder — every claim in this project has terminal output to back it up

---

## How to Reproduce

```bash
# 1. Start ClickHouse in Docker
docker run -d --name clickhouse-server \
  --platform linux/amd64 \
  -p 8123:8123 -p 9000:9000 \
  clickhouse/clickhouse-server:latest

# 2. Connect to the client
docker exec -it clickhouse-server clickhouse-client

# 3. Run any experiment
#    → SQL scripts are embedded in each experiment_report.md
#    → Copy-paste and follow along
```

---

## Key Lessons (The Short Version)

| # | Lesson | The Hard Way We Learned It |
|---|--------|---------------------------|
| 1 | **Background merges are load-bearing** | Disabling them made queries 121× slower |
| 2 | **The sparse index is a hint, not a GPS** | It says *which 8,192-row block*, not *which row* |
| 3 | **An index is only as useful as its data** | Cardinality of 1 = decoration, not optimization |
| 4 | **`read_rows` > wall-clock time** | The "slower" table actually did 195× less wasted work |
| 5 | **Immutability is a feature, not a limitation** | No locks, no WAL, no corruption — but mutations hurt |
| 6 | **Every INSERT creates a folder** | 1 row per INSERT = 1 folder per INSERT = disaster |

---

## Tech Stack

| Component | Version / Tool |
|-----------|---------------|
| ClickHouse | Latest (`clickhouse/clickhouse-server:latest`) |
| Deployment | Docker (linux/amd64) |
| Source Analysis | 305 files from `src/Storages/MergeTree/` |
| Documentation | Markdown + Mermaid diagrams |
| Experiments | ClickHouse SQL client + `system.query_log` |

---

## Authors

**Het Katrodiya & Gaurang Jadav**
Big Data Engineering Project — DA-IICT, Semester 2
