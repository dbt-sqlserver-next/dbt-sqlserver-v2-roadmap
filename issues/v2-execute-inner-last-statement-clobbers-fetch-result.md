---
target_repo: dbt-labs/dbt
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/15765
pr: https://github.com/dbt-labs/dbt/pull/15766
related: ../plan/04-testing-and-validation.md
---

# [v2 Bug] `execute_inner` keeps only the last physical statement's result, even when it is an empty cleanup query

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

`execute_inner` splits a compiled `statement()` block's SQL into physical statements (`splitter.split(sql, adapter_type)`) and executes them in sequence, but the loop only remembers the last statement's response:

```rust
let mut last_batch = None;
for sql in statements {
    last_batch = Some(execute_query_with_retry(..., fetch, ...)?);
}
```

This returns the wrong result whenever a `statement()` block's final physical statement is a no-op cleanup query that runs after the meaningful one.

### Expected Behavior

With `fetch_result=True`, the block returns the result of the statement that produced columns, not the empty result of a trailing cleanup statement.

### Steps To Reproduce

1. Build the SQL Server adapter from #15769. Its `sqlserver__get_test_sql` (vendored from the 1.x adapter) runs `EXEC('create view ...')`, then `select count(*) as failures, ...`, then `EXEC('drop view ...')`. `CREATE VIEW` must be the only statement in its T-SQL batch, so the three are separate physical statements.
2. Run any generic test.
3. `execute_inner` returns the `DROP VIEW`'s empty result (0 rows, 0 columns) instead of the count, so `get_test_results` errors.

### Relevant log output

```shell
Test result table should have 1 row and 3 columns, but got 0 rows and 0 columns
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source with the SQL Server adapter from #15769; SQL Server 2022

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: SQL Server (#15769). The same `sqlserver__get_test_sql` runs every generic test in dbt-sqlserver 1.x. `adapter_impl.rs` is engine code shared by all adapters.

BigQuery, DuckDB and LakeCompute are exempted from splitting (`Bigquery | DuckDB | LakeCompute => vec![sql]`), and the Fabric test macro avoids the multi-statement shape (a single CTE-wrapped `SELECT`, no `CREATE VIEW`), so neither trips this today. Any adapter or macro that emits a multi-statement `fetch_result=True` block ending in a non-`SELECT` statement would.

Fix: track the last statement whose result has columns separately from the last statement, and prefer it when `fetch` is true. The `AdapterResponse` (rows_affected, query_id) still reflects the last statement, preserving multi-statement DML semantics (e.g. delete+insert reporting the insert's rowcount). Implemented in #15766.
