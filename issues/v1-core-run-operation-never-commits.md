---
target_repo: dbt-labs/dbt
branch: 1.13.latest
type: bug
status: open
url: https://github.com/dbt-labs/dbt/issues/16434
related: v1-run-operation-writes-rolled-back.md
---

# `run-operation` never commits, so a macro's writes inside a dbt transaction are rolled back on success

`core/dbt/task/run_operation.py`, `RunOperationTask._run_unsafe` (same on
`1.12.latest` and `1.13.latest`):

```python
with adapter.connection_named("macro_{}".format(macro_name)):
    adapter.clear_transaction()
    res = adapter.execute_macro(
        macro_name, project=package_name, kwargs=macro_kwargs, macro_resolver=self.manifest
    )
```

Nothing commits after `execute_macro`. `statement()` defaults to
`auto_begin=True`, so a macro that writes through it opens a transaction.
Leaving the block calls `release()`, which closes the connection, and `close()`
rolls back any open transaction. The macro succeeds, the command exits 0, and
the write is gone.

```jinja
{% macro probe_insert(tag) %}
  {% call statement('probe') %}insert into dbo.txn_probe values ('{{ tag }}'){% endcall %}
{% endmacro %}
```

```
dbt run-operation probe_insert --args "{tag: x}"   # exit 0

On macro_probe_insert: BEGIN TRANSACTION
insert into dbo.txn_probe values ('x')
On macro_probe_insert: ROLLBACK
On macro_probe_insert: Close
```

The row is not in the table afterwards.

This hits any adapter where `begin` opens a real transaction on the server.
dbt-sqlserver does by default from 1.12, which is how it surfaced.

## Why Postgres doesn't show it

`clear_transaction()` runs `begin()` then `commit()`, and dbt-postgres's
`commit()` sends `COMMIT` as SQL. psycopg2 doesn't parse it, so it still thinks
its implicit transaction is open and never sends another `BEGIN`. Every later
statement in the operation autocommits on the server. Server log for the macro
above:

```
statement: BEGIN
statement: COMMIT
statement: /* ... */ insert into txn_probe values ('stmt_3')
statement: ROLLBACK
```

The write survives, but by accident: a macro that raises partway keeps
whatever ran before the error.

## Fix

Commit on success. `SQLConnectionManager.commit` raises when no transaction is
open, which is the normal case for a macro that only uses `run_query`
(`auto_begin=false`), so check first:

```python
with adapter.connection_named("macro_{}".format(macro_name)):
    adapter.clear_transaction()
    res = adapter.execute_macro(
        macro_name, project=package_name, kwargs=macro_kwargs, macro_resolver=self.manifest
    )
    if adapter.connections.get_thread_connection().transaction_open:
        adapter.connections.commit()
```

A macro that raises still leaves the block through `release()` and rolls back.
`run-operation` already opens a connection for every macro, through
`clear_transaction()`, so the check costs no extra connection.

Measured with dbt-core 1.12.3 and this patch applied, against dbt-sqlserver
1.12.0rc4 on SQL Server 2022 (16.0.4295.3) and dbt-postgres 1.11.0 on
PostgreSQL 16.15:

| | `statement()` write | `run_query` write | macro raises after a write |
|---|---|---|---|
| SQL Server, before | lost | kept | rolled back |
| SQL Server, after | kept | kept | rolled back |
| Postgres, before and after | kept | kept | kept (see above) |

A log-only macro succeeds on both. Snowflake, BigQuery and Spark override
`commit()` with a no-op, so the change does nothing there. That's from their
source; none of the three was measured.

`SQLAdapter.create_schema` and `drop_schema` already commit after
`execute_macro` through `commit_if_has_connection()`. That helper calls
`commit()` unconditionally, so it can't be reused here as is.

Until then, a macro can end with `{% do adapter.commit() %}`, which raises if
the macro opened no transaction.
