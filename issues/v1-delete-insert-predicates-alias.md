---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `delete+insert` predicates can't use `DBT_INTERNAL_DEST`

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

dbt-adapters documents `DBT_INTERNAL_DEST` and `DBT_INTERNAL_SOURCE` as the aliases an `incremental_predicates` entry can use. `sqlserver__get_delete_insert_merge_sql` deletes from the bare target, so the alias doesn't exist:

```sql
{{ config(materialized='incremental', unique_key='id', incremental_strategy='delete+insert',
          incremental_predicates=["DBT_INTERNAL_DEST.id > 0"]) }}
select 1 as id, 'a' as v
```

The first run creates the table; the second fails with `The multi-part identifier "DBT_INTERNAL_DEST.id" could not be bound.` `delete DBT_INTERNAL_DEST from <target> as DBT_INTERNAL_DEST where exists (select 1 from <source> as DBT_INTERNAL_SOURCE where …)` defines both aliases, as `sqlserver__get_incremental_microbatch_sql` already does for its own.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
