---
target_repo: dbt-labs/dbt
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/16504
related: v2-unit-test-actual-cte-nests-model-with-clause-tsql.md
---

# [v2 Bug] Unit tests ignore `unit` materialization and `get_unit_test_sql` overrides

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

A project or adapter that overrides the `unit` materialization or `get_unit_test_sql` has no effect on how v2 runs unit tests, and nothing reports it.

`execute_unit_test_remote_inner` (`dbt-tasks-sa` `runnable/unit_test.rs`) always calls `materialize_unit_test_fast_pass`, which runs the SQL `render_unit_test` builds in Rust with `adapter.execute`. `materialize_unit_test`, the path through the `unit` materialization and so through `get_unit_test_sql`, has no callers.

### Expected Behavior

Either v2 runs unit tests through these overrides as 1.x does, or it tells the user that it doesn't.

### Steps To Reproduce

1. A project with `models/base.sql` (`select 1 as id`), `models/child.sql` (`select id from {{ ref('base') }}`) and one unit test on `child` with `given: ref('base')` rows `[{id: 1}]` and expected rows `[{id: 1}]`.
2. Add either override to `macros/`:
   ```jinja
   {% macro get_unit_test_sql(main_sql, expected_fixture_sql, expected_column_names) -%}
     {{ exceptions.raise_compiler_error("OVERRIDE get_unit_test_sql was called") }}
   {%- endmacro %}
   ```
   ```jinja
   {%- materialization unit, default -%}
     {{ exceptions.raise_compiler_error("OVERRIDE unit materialization was called") }}
   {%- endmaterialization -%}
   ```
3. `dbt build`.

| | No override | `get_unit_test_sql` override | `unit` override |
|---|---|---|---|
| dbt-core 1.12.3 | PASS | ERROR: `OVERRIDE get_unit_test_sql was called` | ERROR: `OVERRIDE unit materialization was called` |
| v2 | Passed | Passed | Passed |

### Relevant log output

```shell
# dbt-core 1.12.3
2 of 3 ERROR child::ut_child ........ [ERROR in 0.08s]
    OVERRIDE get_unit_test_sql was called

# v2
    Passed unit_test ovr_dbt_test__audit.ut_child [3 of 3 in 0.08s]
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` with the SQL Server adapter from #15769; dbt-core 1.12.3 with dbt-sqlserver 1.12.0 for the 1.x column. SQL Server 2022 (16.0.4295.3).

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Measured with the SQL Server adapter from #15769. The code path doesn't depend on the adapter.

Adapter overrides are affected the same way. v2 vendors three, none of which runs:

- `sqlserver__get_unit_test_sql`: SQL Server's case is handled in `render_unit_test` in #15769.
- `fabric__get_unit_test_sql`
- dbt-clickhouse's `unit` materialization: ClickHouse's case was handled in Rust by `7811ca6cd`.

If running unit tests without these overrides is intended, a warning when a project defines either one would stop it being silent, and the vendored adapter overrides could be removed.
