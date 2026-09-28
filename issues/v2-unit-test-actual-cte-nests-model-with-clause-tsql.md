---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# [v2 Bug] Unit tests on SQL Server fail for any model with its own `WITH`: the renderer nests it inside a CTE

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

`render_unit_test` (`dbt-tasks-sa` `renderable/unit_test.rs`) puts the fixtures and the model under test into one flat `WITH` list, and splices the model's SQL in verbatim as the body of `<model>_actual`:

```sql
WITH
    TestDB_raw_raw_stores as (select ... union all select ...),
    TestDB_dbo_stg_locations_expect as (select ... union all select ...),
    TestDB_dbo_stg_locations_actual as (
        with source as (select * from TestDB_raw_raw_stores),
             renamed as (select ... from source)
        select * from renamed
    )
SELECT * FROM (
    (SELECT ..., 'actual' AS actual_or_expected FROM TestDB_dbo_stg_locations_actual)
    UNION ALL
    (SELECT ..., 'expected' AS actual_or_expected FROM TestDB_dbo_stg_locations_expect)
) unit_test_diff
ORDER BY ...
```

T-SQL allows `WITH` only at the start of a statement, so a model with CTEs fails at parse time.

### Expected Behavior

A unit test on a model that starts with `WITH` runs on SQL Server, as it does in 1.x.

### Steps To Reproduce

1. Build the SQL Server adapter from #15769.
2. Run jaffle-shop's `stg_locations` unit test (the model is `with source as (...), renamed as (...) select * from renamed`).
3. It fails with `Incorrect syntax near the keyword 'with'. (ErrorNumber 156)`.

The shapes, run directly on SQL Server 2022:

| Shape | Result |
|---|---|
| `with a as (with p as (select 1 as n) select n from p) select * from a` | Msg 156 |
| `with a as (select * from (with p as (...) select n from p) as w) select * from a` | Msg 156 |
| `select * from (with p as (...) select n from p) as w` | Msg 156 |
| the renderer's shape with the model's CTEs hoisted into the outer list | returns the diff rows |

Wrapping the model in a derived table doesn't help (rows 2–3). The model's CTEs have to be hoisted into the outer list.

### Relevant log output

```shell
Incorrect syntax near the keyword 'with'. (ErrorNumber 156)
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source with the SQL Server adapter from #15769; SQL Server 2022 (16.0.4295.3, Linux)

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: SQL Server (#15769). Fabric likely has the same problem; not measured.

In 1.x, a unit test on a model with its own `WITH` passes: dbt-core 1.12.3, dbt-sqlserver 1.12.0, same server. The 1.x adapter's `sqlserver__get_unit_test_sql` runs the model through a view. That path doesn't exist in v2: the renderer never calls `get_unit_test_sql`.

Fix: for SQL Server, if the rendered model SQL starts with `WITH`, emit its CTEs ahead of `<model>_actual` and use its final `SELECT` as the `_actual` body. Two constraints:

- The model is spliced as raw Jinja and rendered with the whole query, so the hoist has to run on the rendered model SQL, not on `raw_sql`.
- The model's CTE names share a namespace with the fixture CTEs, so a clash needs an error or a rename.
