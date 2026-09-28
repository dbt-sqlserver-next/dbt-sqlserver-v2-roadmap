---
target_repo: dbt-labs/dbt
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/16499
related: v1-core-run-operation-never-commits.md, v1-unit-test-temp-table-left-behind.md
---

# [1.x Bug] The `unit` materialization never commits, so its temp-relation drop is rolled back

### Is this a new bug in dbt-core?

- [X] I believe this is a new bug in dbt-core
- [X] I have searched the existing issues, and I could not find an existing issue for this bug

### Current Behavior

The `unit` materialization (`global_project/macros/materializations/tests/unit.sql`, shipped by dbt-adapters) creates the fixture table through `run_query`, which autocommits. It then runs `statement('main')`, which opens a transaction (`auto_begin=True`), and calls `adapter.drop_relation(temp_relation)` inside it. Nothing commits. The unit-test connection closes with `ROLLBACK`, which undoes the drop, and the empty `<unit_test>__dbt_tmp` table stays in the target schema.

The test passes, and each passing unit test leaves one table behind. This hits any adapter where `begin` opens a real transaction on the server. dbt-sqlserver does by default from 1.12 (`dbt_sqlserver_use_dbt_transactions: true`); with the flag off, the drop autocommits and nothing is left.

Reruns succeed: dbt-sqlserver drops a leftover temp table before creating it, so the "already exists" failure first reported here does not reproduce on 1.12.0rc4 or later.

### Expected Behavior

The temp relation is gone after the unit test finishes.

### Steps To Reproduce

1. dbt-sqlserver 1.12 with `flags: {dbt_sqlserver_use_dbt_transactions: true}` in `dbt_project.yml`.
2. Models `src.sql` (`select 1 as id`) and `m.sql` (`select id * 2 as doubled from {{ ref('src') }}`), and a unit test:

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

3. `dbt run`, then `dbt test --select ut_double`. It passes.
4. `select name from sys.objects where name = 'ut_double__dbt_tmp'` returns the table.

### Relevant log output

```shell
On unit_test.ut16499.m.ut_double: BEGIN TRANSACTION
... unit-test SQL, drop_relation ...
On unit_test.ut16499.m.ut_double: ROLLBACK
On unit_test.ut16499.m.ut_double: Close
```

### Environment

- OS: Linux
- Python: 3.11.16
- dbt: dbt-core 1.12.3, dbt-adapters 1.24.5, dbt-sqlserver 1.12.0 (`v1.12.0rc4-17-g17dd985c`), SQL Server 2022

### Which database adapter are you using with dbt?

other (mention it in "Additional Context")

### Additional Context

Adapter: dbt-sqlserver. The fault is in the shared materialization, not the adapter. It is a separate call path from #16434, which covers `run-operation`.

Fix: commit after the drop.

```jinja
  {% do adapter.drop_relation(temp_relation) %}
  {% do adapter.commit() %}
```

`commit()` does not raise here: `SQLConnectionManager.begin` sets `transaction_open` whether or not the adapter sends a `BEGIN`, and `statement('main')` has just called it. The `test` materialization already commits the same way on its `store_failures` path.

Measured with this change in a project-level `unit` materialization, same environment:

| `dbt_sqlserver_use_dbt_transactions` | before | after |
|---|---|---|
| `true` | `ROLLBACK`, `__dbt_tmp` table left | `COMMIT`, no table left |
| `false` | no table left | no table left |

Not measured on other adapters.
