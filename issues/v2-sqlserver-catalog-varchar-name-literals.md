---
target_repo: dbt-sqlserver-next/dbt-core
type: bug
status: draft
parent: https://github.com/dbt-sqlserver-next/dbt-core/issues/5
related: ./v2-fabric-sp-procedures-name-arguments.md
---

# Catalog queries compare names against `varchar` literals, so names outside the database's code page are lost

`crates/dbt-adapter/src/metadata/sqlserver/mod.rs` builds its three catalog
queries (`build_list_relations_sql`, `build_get_relation_sql`,
`build_columns_sql`) with `SqlLiteralFormatter::format_str`, which for
`SqlServer` emits `'...'` with embedded quotes doubled. An unprefixed literal is
`varchar`, so the name is converted to the code page of the connection's
current database before it is compared with `sys.schemas.name` /
`sys.objects.name` (`sysname`, i.e. `nvarchar(128)`). On a database whose
collation isn't UTF-8, every character outside that code page becomes `?`.

Measured against SQL Server 2022 (16.0.4265.3, Linux), on a
`SQL_Latin1_General_CP1_CI_AS` database, running `build_get_relation_sql`'s
query shape:

| table present | `o.name = '客户'` | `o.name = N'客户'` |
|---|---|---|
| `客户` | 0 rows | 1 row (`TABLE`) |
| `客户` and `??` | 1 row, matching `??` | 1 row, matching `客户` |

`select '客户'` returns `??`.

## Consequence

- `sqlserver_get_relation` (`metadata/get_relation.rs`) returns `None` for an
  existing relation, so dbt treats it as absent.
- When another object's name happens to be the collapsed form, it returns that
  object's relation type instead.
- `SqlServerMetadataAdapter::list_relations` and `list_relations_schemas_inner`
  get the same comparison, so the relation cache and column schemas miss those
  relations too.

Characters that exist in the code page are unaffected: `é`, `ñ` and the rest of
Latin-1 survive `CP1252`. A database with a `_UTF8` collation is also
unaffected. The failure needs a non-UTF-8 collation and a schema or relation
name with characters outside it. CJK, Cyrillic and Greek names on a Latin-1
server are the common cases.

## Fix

Prefix the literal with `N` in all three builders. `format_str` is shared, and
its default arm serves other dialects where `N'...'` means something else or
nothing, so the prefix belongs in the `SqlServer` call sites or a `SqlServer`
arm of `format_str`, not in the default arm. Add a test next to the existing
`o'brien` case in the module's tests.

## v1

v1 has the same bug in its macros, and there it breaks table and incremental
models end to end: `v1-varchar-name-literals-non-codepage.md`, for
dbt-msft/dbt-sqlserver. The v2 fix doesn't need to wait for it.
