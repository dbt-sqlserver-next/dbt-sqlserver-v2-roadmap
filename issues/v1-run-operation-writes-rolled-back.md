---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/862
related: v1-core-run-operation-never-commits.md
---

# With `dbt_sqlserver_use_dbt_transactions` on, `run-operation` silently rolls back a macro's `statement()` writes

A macro run through `dbt run-operation` that writes with
`{% call statement(...) %}` (default `auto_begin=True`) now opens a real
`BEGIN TRANSACTION`. dbt-core never commits after a run-operation, and closing
the connection rolls the transaction back. The command exits 0 with no warning.

```jinja
{% macro probe_insert(tag) %}
  {% call statement('probe') %}insert into dbo.txn_probe values ('{{ tag }}'){% endcall %}
{% endmacro %}
```

| `dbt_sqlserver_use_dbt_transactions` | log | row |
|---|---|---|
| `false` (1.11 default) | insert, `ROLLBACK` (nothing open) | kept |
| `true` (1.12 default) | `BEGIN TRANSACTION`, insert, `ROLLBACK` | lost |

So upgrading to 1.12 silently breaks maintenance macros (cleanup, audit
inserts, custom DDL) that ran fine before. `run_query` is unaffected: it passes
`auto_begin=false`, so each statement autocommits. The adapter's own
`drop_schema_named` is unaffected too, because `adapter.drop_schema()` commits.

Postgres doesn't lose these writes, only because psycopg2 ends up autocommitting
every statement in a run-operation. Root cause is in dbt-core: dbt-labs/dbt#16434.

## Measured

dbt-sqlserver 1.12.0rc4, dbt-core 1.12.3, SQL Server 2022 (16.0.4295.3, Linux).

## Workaround

End the macro with `adapter.commit_if_open()`:

```jinja
{% macro probe_insert(tag) %}
  {% call statement('probe') %}insert into dbo.txn_probe values ('{{ tag }}'){% endcall %}
  {% do adapter.commit_if_open() %}
{% endmacro %}
```

The row persisted with the flag on and off. A macro that opened no transaction
(only `run_query`) runs with it as a no-op, so it's safe to add
unconditionally. Prefer it over `adapter.commit()`, which raises when nothing is
open.
