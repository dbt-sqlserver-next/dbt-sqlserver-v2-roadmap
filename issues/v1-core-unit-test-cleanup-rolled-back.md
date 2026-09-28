---
target_repo: dbt-labs/dbt
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/16499
related: v1-core-run-operation-never-commits.md
---

# The `unit` materialization never commits, so its temp-relation drop is rolled back

`global_project/macros/materializations/tests/unit.sql` (shipped by dbt-adapters) creates the fixture table through `run_query`, which autocommits. It then runs `statement('main')`, which opens a transaction (`auto_begin=True`), and calls `adapter.drop_relation(temp_relation)` inside it. Nothing commits. The unit-test connection closes with `ROLLBACK`, which undoes the drop, and the empty `<unit_test>__dbt_tmp` table stays in the target schema.

```
On unit_test.ut16499.m.ut_double: BEGIN TRANSACTION
... unit-test SQL, drop_relation ...
On unit_test.ut16499.m.ut_double: ROLLBACK
On unit_test.ut16499.m.ut_double: Close
```

The test passes, and each passing unit test leaves one table behind. Reruns succeed: dbt-sqlserver drops a leftover temp table before creating it, so the "already exists" failure first reported here does not reproduce on 1.12.0rc4 or later.

This hits any adapter where `begin` opens a real transaction on the server. dbt-sqlserver does by default from 1.12 (`dbt_sqlserver_use_dbt_transactions: true`). With the flag off, the drop autocommits and nothing is left. It is a separate call path from #16434, which covers `run-operation`.

## Fix

Commit after the drop:

```jinja
  {% do adapter.drop_relation(temp_relation) %}
  {% do adapter.commit() %}
```

`commit()` does not raise here: `SQLConnectionManager.begin` sets `transaction_open` whether or not the adapter sends a `BEGIN`, and `statement('main')` has just called it. The `test` materialization already commits the same way on its `store_failures` path.

Measured with this change in a project-level `unit` materialization, dbt-core 1.12.3, dbt-adapters 1.24.5, dbt-sqlserver 1.12.0 (`v1.12.0rc4-17-g17dd985c`), SQL Server 2022:

| `dbt_sqlserver_use_dbt_transactions` | before | after |
|---|---|---|
| `true` | `ROLLBACK`, `__dbt_tmp` table left | `COMMIT`, no table left |
| `false` | no table left | no table left |

Not measured on other adapters.
