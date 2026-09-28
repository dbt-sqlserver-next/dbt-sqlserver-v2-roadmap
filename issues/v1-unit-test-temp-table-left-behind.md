---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
url:
related: v1-core-unit-test-cleanup-rolled-back.md
---

# With `dbt_sqlserver_use_dbt_transactions` on, every passing unit test leaves a `__dbt_tmp` table behind

With the flag on (the 1.12 default), each unit test leaves an empty `<unit_test>__dbt_tmp` table in the target schema. The test passes and dbt reports nothing. dbt-core's `unit` materialization drops the table inside the transaction `statement('main')` opens, never commits, and closing the connection rolls the drop back. Root cause and fix are in dbt-core: dbt-labs/dbt#16499.

| `dbt_sqlserver_use_dbt_transactions` | log | `__dbt_tmp` table |
|---|---|---|
| `false` (1.11 default) | drop autocommits | dropped |
| `true` (1.12 default) | `BEGIN TRANSACTION`, drop, `ROLLBACK` | left behind |

Reruns still pass: the adapter drops a leftover temp table before creating it, so the table is replaced, not duplicated. What remains is one empty table per unit test in the schema.

## Steps to reproduce

`dbt_project.yml`:

```yaml
flags:
  dbt_sqlserver_use_dbt_transactions: true
```

Models `src.sql` (`select 1 as id`) and `m.sql` (`select id * 2 as doubled from {{ ref('src') }}`), and:

```yaml
unit_tests:
  - name: ut_double
    model: m
    given:
      - input: ref('src')
        rows: [{id: 2}]
    expect:
      rows: [{doubled: 4}]
```

`dbt run`, then `dbt test --select ut_double`. It passes, and `select name from sys.objects where name = 'ut_double__dbt_tmp'` returns the table.

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
On unit_test.ut16499.m.ut_double: BEGIN TRANSACTION
... unit-test SQL, drop_relation ...
On unit_test.ut16499.m.ut_double: ROLLBACK
On unit_test.ut16499.m.ut_double: Close
```

## Workaround

Copy dbt-core's `unit` materialization into the project as `materialization unit, adapter='sqlserver'` and add a commit after the drop:

```jinja
  {% do adapter.drop_relation(temp_relation) %}
  {% do adapter.commit() %}
```

With the flag on, the log ends in `COMMIT` and no table is left; with it off, the run is unchanged.
