---
target_repo: dbt-labs/dbt
branch: 1.13.latest
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/16434
related: v1-run-operation-writes-rolled-back.md
---

# [1.x Bug] `run-operation` never commits, so writes inside a dbt transaction are rolled back on success

### Is this a new bug in dbt-core?

- [X] I believe this is a new bug in dbt-core
- [X] I have searched the existing issues, and I could not find an existing issue for this bug

### Current Behavior

`core/dbt/task/run_operation.py` never commits, on either path (same on `1.12.latest` and `1.13.latest`). `RunOperationTask._run_unsafe`:

```python
with adapter.connection_named("macro_{}".format(macro_name)):
    adapter.clear_transaction()
    res = adapter.execute_macro(
        macro_name, project=package_name, kwargs=macro_kwargs, macro_resolver=self.manifest
    )
```

`_run_unsafe_sql`, for `run-operation --sql`:

```python
with adapter.connection_named("inline_query"):
    adapter.clear_transaction()
    response, _ = adapter.execute(sql, auto_begin=True, fetch=False)
```

`statement()` defaults to `auto_begin=True`, so a macro that writes through it opens a transaction, and `--sql` always opens one. Leaving the block calls `release()`, which closes the connection, and `close()` rolls back any open transaction. The command exits 0 and the write is gone.

This hits any adapter where `begin` opens a real transaction on the server. dbt-sqlserver does by default from 1.12, which is how it surfaced.

### Expected Behavior

A run-operation that succeeds keeps its writes; one that raises rolls them back.

### Steps To Reproduce

1. dbt-sqlserver with `flags: {dbt_sqlserver_use_dbt_transactions: true}` and a table `dbo.txn_probe (tag varchar(50))`.
2. A macro:

```jinja
{% macro probe_insert(tag) %}
  {% call statement('probe') %}insert into dbo.txn_probe values ('{{ tag }}'){% endcall %}
{% endmacro %}
```

3. `dbt run-operation probe_insert --args "{tag: x}"` exits 0.
4. `dbt run-operation --sql "insert into dbo.txn_probe values ('y')"` exits 0.
5. Neither row is in the table.

### Relevant log output

```shell
On macro_probe_insert: BEGIN TRANSACTION
insert into dbo.txn_probe values ('x')
On macro_probe_insert: ROLLBACK
On macro_probe_insert: Close
```

### Environment

- OS: Linux
- Python: 3.11
- dbt: dbt-core 1.12.3, dbt-sqlserver 1.12.0rc4 and 1.12.0, SQL Server 2022 (16.0.4295.3); dbt-postgres 1.11.0, PostgreSQL 16.15

### Which database adapter are you using with dbt?

postgres, other (mention it in "Additional Context")

### Additional Context

Adapter: dbt-sqlserver. dbt-sqlserver 1.12.0 works around it by committing on connections named `macro_*` and `inline_query` (dbt-msft/dbt-sqlserver#866); that stopgap was disabled for the measurements below.

**Fix.** Commit on success, in both blocks. `SQLConnectionManager.commit` raises when no transaction is open, which is the normal case for a macro that only uses `run_query` (`auto_begin=false`), so check first:

```python
with adapter.connection_named("macro_{}".format(macro_name)):
    adapter.clear_transaction()
    res = adapter.execute_macro(
        macro_name, project=package_name, kwargs=macro_kwargs, macro_resolver=self.manifest
    )
    if adapter.connections.get_thread_connection().transaction_open:
        adapter.connections.commit()
```

and the same check after `adapter.execute(...)` in `_run_unsafe_sql`. A run-operation that raises still leaves the block through `release()` and rolls back. `clear_transaction()` already opens the connection, so the check costs no extra one.

`SQLAdapter.create_schema` and `drop_schema` already commit after `execute_macro` through `commit_if_has_connection()`. That helper calls `commit()` unconditionally, so it can't be reused here as is.

Measured with dbt-core 1.12.3 and this patch applied:

| | `statement()` write | `run_query` write | macro raises after a write | `--sql` write | `--sql` batch fails after a write |
|---|---|---|---|---|---|
| SQL Server, before | lost | kept | rolled back | lost | rolled back |
| SQL Server, after | kept | kept | rolled back | kept | rolled back |
| Postgres, before and after | kept | kept | kept | not measured | not measured |

A log-only macro succeeds on both. Snowflake, BigQuery and Spark override `commit()` with a no-op, so the change does nothing there. That's from their source; none of the three was measured.

**Why Postgres doesn't show it.** `clear_transaction()` runs `begin()` then `commit()`, and dbt-postgres's `commit()` sends `COMMIT` as SQL. psycopg2 doesn't parse it, so it still thinks its implicit transaction is open and never sends another `BEGIN`. Every later statement in the operation autocommits on the server:

```
statement: BEGIN
statement: COMMIT
statement: /* ... */ insert into txn_probe values ('stmt_3')
statement: ROLLBACK
```

The write survives, but by accident: a macro that raises partway keeps whatever ran before the error.

Workaround: end the macro with `{% do adapter.commit() %}`, which raises if the macro opened no transaction.
