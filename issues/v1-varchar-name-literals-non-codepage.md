---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: v2-sqlserver-catalog-varchar-name-literals.md
---

# Relation names outside the database's code page break table and incremental models: macros pass them as `varchar` literals

Many macros put a schema or relation name in an unprefixed `'...'` literal,
e.g. `sqlserver__rename_relation`:

```jinja
EXEC sp_rename '{{ escape_single_quotes(from_relation.include(database=False)) }}', '{{ escape_single_quotes(to_relation.identifier) }}'
```

An unprefixed literal is `varchar`, so on a database whose collation isn't
UTF-8 every character outside its code page becomes `?` before `sp_rename`,
`OBJECT_ID()` or a `sys.*.name` comparison sees it. Identifiers in the statement
text itself aren't affected, so the `CREATE` succeeds and the lookups that
follow miss.

## Measured

SQL Server 2022 (16.0.4295.3, Linux), database collation
`SQL_Latin1_General_CP1_CS_AS`, dbt-core 1.12.3, dbt-sqlserver at `10a5899`
(master, after 1.12.0rc4). Models are `select 1 as id`:

| Model | As is | With `N'...'` |
|---|---|---|
| table `客户`, run 1 | `No item by the name of '"dspine_scratch"."??__dbt_tmp"' could be found (15225)` from `sqlserver__rename_relation` | OK, and on runs 2 and 3 |
| incremental `增量`, run 2 | `Incorrect syntax near the keyword 'when'. (156)` | OK |
| view `视图` | OK | — |

The incremental case: `sqlserver__get_columns_in_relation` filters on
`object_id('"TestDB"."dspine_scratch"."增量"')`, which is `NULL` once the name
becomes `??`. That returns no columns, so the `merge` renders
`when matched then update set` with an empty list. The "With `N'...'`" column
adds the prefix to those two macros only.

## Fix

Prefix every name literal with `N`. Beyond the two above, the unprefixed ones
are in `adapters/`: `apply_denies.sql`, `apply_grants.sql`, `apply_masks.sql`,
`catalog.sql`, `columns.sql` (the column `sp_rename`), `indexes.sql`,
`metadata.sql`, `relation.sql` and `schema.sql`; in
`relations/table/create.sql`, `materializations/models/table/columns_spec_ddl.sql`
and `utils/get_tables_by_pattern.sql`. `columns.sql`'s `sp_describe_first_result_set`
and `persist_docs.sql` already use `N'...'`.

A functional test with a model named outside Latin-1 would cover both paths
above.
