# ClickHouse MergeTree Engine — Big Data Engineering Project

## Project Overview
A deep-dive systems engineering study of ClickHouse's MergeTree storage engine.
Covers architecture, execution paths, codebase analysis, and real experiments.

##  ClickHouse Versin ##

SELECT version(); -- 24.x.x
## Repository Structure

| Folder | Contents |
|--------|----------|
| `/report` | Full markdown report (12 sections) |
| `/experiments` | SQL scripts + results for all 3 experiments |
| `/screenshots` | Terminal/query output screenshots |
| `/code-notes` | Key ClickHouse source file annotations |
| `/diagrams` | Architecture and flow diagrams |

## Experiments Conducted
- **Exp 1:** `index_granularity` impact on query performance
- **Exp 2:** Small vs large dataset — merge behavior observation
- **Exp 3:** ClickHouse code modification + rebuild + behavior change

## How to Reproduce
```bash
docker run -d --name clickhouse-server \
  --platform linux/amd64 \
  -p 8123:8123 -p 9000:9000 \
  clickhouse/clickhouse-server:latest
```

## Author
Het Katrodiya, Gaurang Jadav -Big Data Engineering Project
