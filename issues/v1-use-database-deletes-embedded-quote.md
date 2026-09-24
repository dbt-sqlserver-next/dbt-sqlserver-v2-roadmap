---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: v1-quoting-alignment-with-v2.md
---

# `get_use_database_sql` deletes an embedded `"` instead of escaping it, so `USE` can select a different database than the rest of the run

`dbt/include/sqlserver/macros/adapters/metadata.sql`:

```jinja
{%- macro sqlserver__get_use_database_sql(database) -%}
  USE {{ adapter.quote(database | replace('"', '')) }};
{%- endmacro -%}
```

The filter runs first, so `adapter.quote()` never sees the character it escapes.
For a database named `we"ird`:

| path | emits |
|---|---|
| `SQLServerAdapter.quote`, `{{ relation }}` | `"we""ird"` |
| `sqlserver__get_use_database_sql` | `"weird"` |

It's the only `replace('"', '')` left under `dbt/`.

## Measured

SQL Server 2022 (16.0.4265.3, Linux), against v1.12.0rc4:

```
USE "we""ird";   -> Changed database context to 'we"ird'.
USE "weird";     -> Changed database context to 'weird'.     (if it exists)
USE "weird";     -> Msg 911: Database 'weird' does not exist. (if it doesn't)
```

If `weird` exists, the run silently continues in the wrong database while every
statement rendered through the relation still targets `"we""ird"`. If it
doesn't, the error names a database the user never configured.

The name comes from `profiles.yml` or a `database:` config, so a `"` in it is
unusual, but the wrong-database case gives no error.

## Origin

`19c32a1` copied Fabric's `USE [{{database | replace('"', '')}}];`. Inside
brackets a `"` is an ordinary character, so deleting it was wrong but harmless.
#795 switched to `adapter.quote()` and kept the filter, which made it matter.

## Fix

```jinja
  USE {{ adapter.quote(database) }};
```

`test_quote.py::test_quote_escapes_embedded_delimiters` already covers
`adapter.quote()`. `test_macros_do_not_hand_format_identifiers` misses this case
because the name does reach `adapter.quote()`; a `HAND_QUOTING` pattern for
`replace` of a quote character would catch it.

## References

- https://learn.microsoft.com/sql/relational-databases/databases/database-identifiers
