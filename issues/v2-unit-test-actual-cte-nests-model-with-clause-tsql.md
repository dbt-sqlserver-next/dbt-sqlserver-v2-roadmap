---
target_repo: dbt-labs/dbt
type: bug
status: closed
resolution: not filed; fixed on the port by dbt-sqlserver-next/dbt-core#26
related: ../plan/04-testing-and-validation.md
---

# [v2 Bug] Unit tests never call the adapter's `get_unit_test_sql`, so SQL Server's 1.x override is lost and models with their own `WITH` fail

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

In 1.x, the `unit` materialization renders the test through `get_unit_test_sql`, which adapters override: dbt-sqlserver (`sqlserver__get_unit_test_sql`), dbt-fabric (`fabric__get_unit_test_sql`) and dbt-clickhouse (its own `unit` materialization). v2 vendors all three, but never calls them. `execute_unit_test_remote_inner` (`dbt-tasks-sa` `runnable/unit_test.rs`) always runs `materialize_unit_test_fast_pass`, which executes the SQL built by `render_unit_test` with `adapter.execute`. `materialize_unit_test`, the path through the `unit` materialization, has no callers.

ClickHouse's case was re-implemented in Rust (`7811ca6cd`, #16153). SQL Server's wasn't, and the shape its override exists to avoid now fails:

`render_unit_test` (`dbt-tasks-sa` `renderable/renderable/unit_test.rs`) puts the fixtures and the model under test into one flat `WITH` list, and splices the model's SQL in verbatim as the body of `<model>_actual`:

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

Adapter: SQL Server (#15769). In 1.x, a unit test on a model with its own `WITH` passes: dbt-core 1.12.3, dbt-sqlserver 1.12.0, same server. `sqlserver__get_unit_test_sql` creates the model and the expected rows as views (`EXEC('create view … as <sql>')`, where a leading `WITH` is legal), selects the diff from them, and drops them. The same `EXEC('create view')`/select/drop shape already runs every SQL Server generic test on v2 (`sqlserver__get_test_sql`), so it works on v2's execution path.

Fix: implement the 1.x adapter override on v2. For an adapter that defines `<adapter>__get_unit_test_sql`, render the diff through it, the way `unit` does in 1.x, passing:

- `main_sql`: the model's rendered SQL with the fixtures injected as CTEs. When the model starts with `WITH`, they have to be spliced into its list; `inject_ctes_into_existing_with` (`dbt-jinja-utils`) already does that for ephemeral models.
- `expected_fixture_sql` and the quoted expected column names, which `render_unit_test` already builds.

`unit_test.rs` already branches per adapter (DuckDB and ClickHouse schema inference, Snowflake/BigQuery/Databricks typing), so the branch fits the file. Fabric's `fabric__get_unit_test_sql` is dead in v2 too; not measured on Fabric.
