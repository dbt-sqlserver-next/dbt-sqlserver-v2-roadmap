---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: https://github.com/dbt-msft/dbt-sqlserver/issues/166
---

# [v2 Bug] The ephemeral select wrapper's derived table has no alias, which SQL Server requires

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

When a model that refs an ephemeral model doesn't start with its own `WITH`, `inject_and_persist_ephemeral_models` (`dbt-jinja-utils` `utils.rs`) wraps it:

```sql
with __dbt__cte__eph as (
select 7 as n
)
--EPHEMERAL-SELECT-WRAPPER-START
select * from (
select n from __dbt__cte__eph
--EPHEMERAL-SELECT-WRAPPER-END
)
```

The derived table has no alias. T-SQL requires one, so on SQL Server every such model fails:

```
Incorrect syntax near ';'. (ErrorNumber 102, State 1, Class 15, LineNo 10)
```

The same fallback runs for every adapter. PostgreSQL before 16 also rejects a derived table without an alias.

### Expected Behavior

The model builds, as it does in dbt 1.x, where the ephemeral CTE is prepended with no wrapper.

### Steps To Reproduce

1. With the SQL Server adapter from #15769, add `models/eph.sql`:
   ```sql
   {{ config(materialized='ephemeral') }}
   select 7 as n
   ```
   and `models/use_eph.sql`:
   ```sql
   select n from {{ ref('eph') }}
   ```
2. `dbt run -s use_eph`

### Relevant log output

```shell
[error] [DbDriverFailed (dbt1308)]: Database Error in model use_eph (target/run/v2sync/models/use_eph.sql)
  [mssql] Could not execute query: Incorrect syntax near ';'. (ErrorNumber 102, State 1, Class 15, LineNo 10)
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source (`3d61704d4`) with the SQL Server adapter from #15769; SQL Server 2022 (16.0.4295.3)

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: SQL Server (#15769). dbt-sqlserver 1.12.0 runs ephemeral models ([dbt-sqlserver#166](https://github.com/dbt-msft/dbt-sqlserver/issues/166)), except one that starts with its own `WITH`, which T-SQL can't nest inside a CTE.

A fix is to alias the derived table (`) __dbt_ephemeral_select`, without `as`). The SQL comparison in `dbt-adapter` `sql/tokenizer.rs` strips the wrapper by its markers and asserts that `)` follows `--EPHEMERAL-SELECT-WRAPPER-END`, so it would also need to skip the alias.
