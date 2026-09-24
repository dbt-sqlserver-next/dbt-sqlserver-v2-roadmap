---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# Unit tests on SQL Server fail for any model with its own `WITH`: the renderer nests it inside a CTE

`render_unit_test` (`dbt-tasks-sa` `renderable/unit_test.rs`) puts the fixtures
and the model under test into one flat `WITH` list, and splices the model's SQL
in verbatim as the body of `<model>_actual`:

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

T-SQL allows `WITH` only at the start of a statement, so a model with CTEs
fails at parse time. With the SQL Server adapter from #15769, jaffle-shop's
`stg_locations` unit test fails with `Incorrect syntax near the keyword 'with'.
(ErrorNumber 156)`.

## Measured

SQL Server 2022 (16.0.4295.3, Linux):

| Shape | Result |
|---|---|
| `with a as (with p as (select 1 as n) select n from p) select * from a` | Msg 156 |
| `with a as (select * from (with p as (...) select n from p) as w) select * from a` | Msg 156 |
| `select * from (with p as (...) select n from p) as w` | Msg 156 |
| the renderer's shape with the model's CTEs hoisted into the outer list | returns the diff rows |

Wrapping the model in a derived table doesn't help (rows 2–3). The model's CTEs
have to be hoisted into the outer list.

## Fix

For SQL Server (and likely Fabric, which isn't measured here), if the rendered
model SQL starts with `WITH`, emit its CTEs ahead of `<model>_actual` and use
its final `SELECT` as the `_actual` body. Two constraints:

- The model is spliced as raw Jinja and rendered with the whole query, so the
  hoist has to run on the rendered model SQL, not on `raw_sql`.
- The model's CTE names share a namespace with the fixture CTEs, so a clash
  needs an error or a rename.

The adapter's `sqlserver__get_unit_test_sql`, which runs the model through a
view, isn't on this path: the renderer never calls `get_unit_test_sql`.
