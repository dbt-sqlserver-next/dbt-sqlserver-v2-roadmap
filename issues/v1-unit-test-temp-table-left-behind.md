---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/874
related: v1-core-unit-test-cleanup-rolled-back.md, v1-run-operation-writes-rolled-back.md
---

# With `dbt_sqlserver_use_dbt_transactions` on, every unit test leaves a `__dbt_tmp` table behind

With the flag on (the 1.12 default), each unit test leaves an empty `<unit_test>__dbt_tmp` table in the target schema, whether it passes or fails. dbt reports nothing. dbt-core's `unit` materialization drops the table inside the transaction `statement('main')` opens and never commits, so closing the connection rolls the drop back. Root cause: dbt-labs/dbt#16499.

| `dbt_sqlserver_use_dbt_transactions` | log | `__dbt_tmp` table |
|---|---|---|
| `false` (1.11 default) | drop autocommits | dropped |
| `true` (1.12 default) | `BEGIN TRANSACTION`, drop, `ROLLBACK` | left behind |

Reruns still pass: the adapter drops a leftover temp table before creating it, so the table is replaced, not duplicated.

## Steps to reproduce

`dbt_project.yml`:

```yaml
flags:
  dbt_sqlserver_use_dbt_transactions: true
```

Models `src.sql` (`select 1 as id`) and `m.sql` (`select id * 2 as doubled from {{ ref('src') }}`), and:

```yaml
unit_tests:
  - name: ut_ok
    model: m
    given: [{input: ref('src'), rows: [{id: 2}]}]
    expect: {rows: [{doubled: 4}]}
  - name: ut_diff
    model: m
    given: [{input: ref('src'), rows: [{id: 2}]}]
    expect: {rows: [{doubled: 5}]}
```

`dbt run`, then `dbt test --select ut_ok ut_diff`. `ut_ok` passes and `ut_diff` fails; both `ut_ok__dbt_tmp` and `ut_diff__dbt_tmp` remain in `sys.objects`.

## Environment

- Database: SQL Server 2022 (Linux container)
- Backend: mssql-python
- Authentication, ODBC driver, OS: SQL login, ODBC Driver 18 for SQL Server, Linux

`dbt --version`:
```text
dbt-core 1.12.3, dbt-adapters 1.24.5, dbt-sqlserver 1.12.0 (v1.12.0rc4-17-g17dd985c)
```

## Log excerpt

```text
On unit_test.proj.m.ut_ok: BEGIN TRANSACTION
... unit-test SQL, drop_relation ...
On unit_test.proj.m.ut_ok: ROLLBACK
On unit_test.proj.m.ut_ok: Close
```

## Fix

dbt-core runs each unit test inside `connection_named(node.unique_id)`, so the unit-test connection is named `unit_test.<project>.<model>.<test>`. Adding `"unit_test."` to `SQLServerAdapter._RUN_OPERATION_CONNECTION_PREFIXES` extends the #866 stopgap to it: `connection_named` then runs `commit_if_open()` when the unit test finishes without raising. Measured with that change, both `__dbt_tmp` tables are gone after the run above, and `ut_ok` still passes and `ut_diff` still fails. The constant's name and the `connection_named` docstring would need to cover unit tests too.

## Workaround

Copy dbt-core's `unit` materialization into the project as `materialization unit, adapter='sqlserver'` and add a commit after the drop:

```jinja
  {% do adapter.drop_relation(temp_relation) %}
  {% do adapter.commit() %}
```
