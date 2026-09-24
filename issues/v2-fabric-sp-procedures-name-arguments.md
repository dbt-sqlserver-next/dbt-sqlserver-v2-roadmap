---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ../plan/02-implementation-steps.md
---

# Fabric metadata passes relation names to `sp_tables` / `sp_columns` unquoted and as `LIKE` patterns

Three calls in `dbt-adapter` build `EXEC` statements from relation names:

| Call site | Statement | Arguments |
|---|---|---|
| `metadata/fabric/mod.rs` `list_relations` (`list_relations_without_caching`) | `sp_tables @table_qualifier, @table_owner` | bare |
| `metadata/fabric/mod.rs` `FabricMetadataAdapter::list_relations_schemas_inner` (unit-test `given` schemas) | `sp_columns @table_qualifier, @table_owner, @table_name` | bare |
| `metadata/get_relation.rs` `fabric_get_relation` | `sp_tables @table_qualifier, @table_owner, @table_name` | `SqlLiteralFormatter::format_str` |

Two separate problems:

1. **Bare arguments.** `EXEC` accepts an unquoted argument only if it's a
   regular identifier. Anything else fails to parse, and a `'` opens a string
   literal.
2. **Patterns.** `@table_owner` and `@table_name` are `LIKE` patterns, so `_`,
   `%` and `[` in a name match other objects. This includes the quoted call.

## Measured

SQL Server 2022 (16.0.4295.3, Linux), running the strings these `format!`s
produce. I have no Fabric warehouse to test against.

Bare vs quoted, `sp_columns` on `dbo.<name>`:

| Name | Bare, as emitted | `N'...'` |
|---|---|---|
| `plain` | 1 row | 1 row |
| `order-items` | Msg 156 | 1 row |
| `2024_sales` | Msg 102 | 1 row |
| `order` | Msg 156 | 1 row |
| `my table` | Msg 156 | 1 row |
| `o'brien` | Msg 105, unclosed quotation mark | 1 row |

A schema `my-schema` in `list_relations` is Msg 102.

Patterns, with schemas `ab_c` and `abXc`, and tables `ab`, `a[b`, `50%` and
`50x` in `ab_c`:

| Arguments | Rows |
|---|---|
| `sp_tables @table_owner = ab_c` | `ab_c.*` and `abXc.*` |
| `sp_columns @table_owner = N'ab_c', @table_name = N't_'` | columns of `ab_c.t1` and `abXc.t2` |
| `sp_tables @table_name = N'a[b'` | `ab`, not `a[b` |
| `sp_tables @table_name = N'50%'` | `50%` and `50x` |
| same, with `[_]`, `[[]`, `[%]` escapes | exactly the named object |

Consequences:

- `fabric_get_relation` errors when more than one row comes back ("Did not find
  'TABLE_TYPE' for a relation").
- `build_schema_from_sp_columns` appends every returned row, so a unit test's
  `given` schema silently gains the other table's columns.
- `list_relations` fails for any schema that isn't a regular identifier.

## Fix

Quote every argument with `format_str` plus an `N` prefix. A plain `'...'` is
`varchar`: on a code page 1252 database `'客户'` finds 0 rows in both
procedures, while `N'客户'` finds 1. Then escape the two pattern arguments by
wrapping `[`, `_` and `%` in brackets.

`@fUsePattern = 0` isn't a general alternative. `sp_tables` accepts it, but
with `@table_name = NULL` (as `list_relations` calls it) it returns 0 rows, and
`sp_columns` has no such parameter ("@fUsePattern is not a parameter for
procedure sp_columns").
