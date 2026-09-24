---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/02-implementation-steps.md
---

# `check_schema_exists` and `get_relation_last_modified` ignore their database argument and answer about whichever database `USE` last selected

Both macros in `dbt/include/sqlserver/macros/adapters/metadata.sql` take an
`information_schema` argument, discard it, and read unqualified `sys.schemas` /
`sys.objects`. `sys.*` is scoped to the database, so the answer depends on the
connection's current database, which earlier macros set with `USE` and never
restore. The other 34 `get_use_database_sql` call sites emit `USE` first; these
two don't.

```jinja
{% macro sqlserver__check_schema_exists(information_schema, schema) -%}
  ...
    SELECT count(*) as schema_exist FROM sys.schemas WHERE name = '{{ schema }}' ...
```

## Measured

SQL Server 2022 (16.0.4265.3, Linux). `xdb_a` holds `only_in_a.orders`; the
connection is on `xdb_b`; dbt asks about `xdb_a`:

| | on `xdb_b` | after an unrelated `USE [xdb_a]` | truth |
|---|---|---|---|
| `check_schema_exists('xdb_a', 'only_in_a')` | 0 | 1 | 1 |
| `get_relation_last_modified(xdb_a.only_in_a.orders)` | 0 rows | 1 row | 1 row |

`USE` persists for every later statement on the connection (dbt reuses one per
thread), and survives a rollback: `begin transaction; USE [other]; rollback`
leaves the connection on `other`.

## Impact

`dbt source freshness` on a source without `loaded_at_field` goes through
`get_relation_last_modified`, and sources in another database are supported
(`tests/functional/adapter/mssql/test_cross_db.py`). The answer depends on which
model last ran on that thread, so it won't reproduce reliably. An empty result
reads as "no freshness information", not an error.

## Fix

Qualify the reads instead of relying on connection state:

```jinja
{% set db = adapter.quote(information_schema.database) %}
... FROM {{ db }}.sys.schemas WHERE ...
```

and the same prefix on `sqlserver__get_relation_last_modified`'s `sys.objects` /
`sys.schemas`. `calculate_freshness_from_metadata_batch` groups relations by
`information_schema`, so one call is always one database.

Adding `get_use_database_sql` also gives the right answer, but it turns a pure
read (`check_schema_exists` runs with `auto_begin=False`) into one that changes
the database for whatever runs next, which is how this bug happens.

`USE` itself has to stay: `CREATE VIEW`, `CREATE PROCEDURE` (error 166) and
`CREATE SCHEMA` (error 102) reject a database prefix, and `EXEC('...')` bodies
resolve in the current database. The consistent rule is three-part names
wherever a statement accepts them and `USE` only where it doesn't. Moving the
other call sites is a separate PR. Before one, check Azure SQL Database, which
doesn't support cross-database three-part names.

## Other macros to check

These read `sys.*` or `INFORMATION_SCHEMA.*` without emitting `USE`; each is
correct only if every caller emits it first:

```
adapters/apply_denies.sql    sqlserver__get_show_deny_sql
                             sqlserver__get_existing_principals
adapters/apply_grants.sql    sqlserver__get_show_grant_sql
adapters/apply_masks.sql     sqlserver__get_show_mask_sql
                             sqlserver__get_mask_index_key_columns
                             sqlserver__get_unmaskable_columns
adapters/persist_docs.sql    sqlserver__alter_relation_comment
                             sqlserver__alter_column_comment
utils/                       sqlserver__get_tables_by_pattern_sql
```

The `indexes.sql` and `relations/table/create.sql` helpers of the same shape are
only called from bodies that emit `USE`.
