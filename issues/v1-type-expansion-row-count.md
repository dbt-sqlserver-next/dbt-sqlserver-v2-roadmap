---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: v1-column-ddl-on-schema-change.md
---

# `expand_column_types` counts rows before it knows it needs to

With `dbt_sqlserver_enable_safe_type_expansion` on and a positive `column_type_expansion_max_rows` (default 1,000,000), `_safe_expansion_allowed` runs `SELECT COUNT_BIG(*)` on the target on every incremental and snapshot run, including runs where no column differs.

| Target | `COUNT_BIG(*)` |
|---|---|
| 20M-row heap | 0.15 s |
| 20M-row clustered columnstore | 0.003 s |

Counting only when a column qualifies for a cross-family promotion skips it on every run that has none. The gain is small: it applies to opt-in users with rowstore or heap targets.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
