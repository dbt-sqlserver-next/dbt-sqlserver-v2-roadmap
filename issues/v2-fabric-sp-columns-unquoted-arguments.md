---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ./v2-fabric-sp-tables-like-pattern.md
---

# Fabric's column introspection pastes relation names into `EXEC sp_columns` unquoted

`FabricMetadataAdapter::list_relations_schemas_inner`
(`crates/dbt-adapter/src/metadata/fabric/mod.rs`) builds its query as:

```rust
let sql = format!(
    "EXEC sp_columns @table_qualifier={}, @table_owner={}, @table_name={}",
    relation.database_as_str()?,
    relation.schema_as_str()?,
    relation.identifier_as_str()?,
);
```

`database_as_str` and its siblings return the stored name as-is, so each
argument reaches T-SQL as a bare word. T-SQL accepts a bare procedure argument
only when it happens to be a regular identifier, so any name that isn't one
fails to parse, and a `'` in a name opens a string literal.

`fabric_get_relation` (`crates/dbt-adapter/src/metadata/get_relation.rs`)
passes the same three values through `SqlLiteralFormatter::format_str`; this
call site doesn't.

Measured against SQL Server 2022 (16.0.4265.3, Linux), running the exact
string the `format!` above produces. `sp_columns` is the same procedure there;
I have no Fabric warehouse to test against.

## Summary

Each table exists in `dbo` of the connection's database:

| name | bare, as emitted today | as an `N'...'` literal |
|---|---|---|
| `plain` | 1 row | 1 row |
| `order-items` | `Msg 156: Incorrect syntax near the keyword 'order'` | 1 row |
| `2024_sales` | `Msg 102: Incorrect syntax near '_sales'` | 1 row |
| `order` | `Msg 156: Incorrect syntax near the keyword 'order'` | 1 row |
| `my table` | `Msg 156: Incorrect syntax near the keyword 'table'` | 1 row |
| `o'brien` | `Msg 105: Unclosed quotation mark after the character string 'brien` | 1 row |

A database named `spc-repro` fails the same way as the qualifier
(`Msg 102: Incorrect syntax near '-'`).

## Consequences

The query errors for any relation whose database, schema or name is a reserved
word, starts with a digit, or contains a space, `-` or other character a regular
identifier can't hold. Its one caller is `MetadataAdapter::list_relations_sdf_schemas`,
which `hydrate_unit_test_relation_schema` (`renderable/unit_test.rs`) uses to
resolve the schema of a unit test's `given` relations, so on Fabric those unit
tests fail. `get_columns_in_relation` isn't affected: `fabric__get_columns_in_relation`
is a Jinja macro with its own query.

In the `'` case, the rest of the statement is read as a string literal, and
text after a second `'` would run as T-SQL. The names come from the project's
own models and sources, so this is a correctness bug more than an injection
surface, but nothing escapes them.

## Fix

Pass each value through `SqlLiteralFormatter::format_str`, the same as
`fabric_get_relation`. That fixes every row in the table above except for names
outside the database's code page: `format_str` emits a plain `'...'` literal,
which is `varchar`, and `sp_columns` takes `nvarchar`. Measured on a
`SQL_Latin1_General_CP1_CI_AS` database with a table named `客户`:

| literal | `sp_columns` | `sp_tables` |
|---|---|---|
| `'客户'` | 0 rows | 0 rows |
| `N'客户'` | 1 row | 1 row |

`'客户'` is converted to the database's code page before the procedure sees it,
becoming `'??'`. An `N` prefix on the literal fixes both call sites, including
the `sp_tables` call in `fabric_get_relation`.

The LIKE-pattern problem in `v2-fabric-sp-tables-like-pattern.md` is on the same
`sp_columns` line, and its fix (`@fUsePattern = 0`) lands in the same
`format!`, so one PR can take both.

## References

- `sp_columns`:
  https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-columns-transact-sql
- Regular identifier rules:
  https://learn.microsoft.com/sql/relational-databases/databases/database-identifiers
- Found while porting dbt-sqlserver to v2: dbt-labs/dbt#15714
