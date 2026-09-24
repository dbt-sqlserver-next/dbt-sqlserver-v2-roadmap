---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: https://github.com/dbt-sqlserver-next/dbt-core/pull/21
---

# Unit-test fixtures cast to `<type> NOT NULL` when the `given` relation's column is NOT NULL

`1330d0219` added a `nullable` parameter to `TypeOps::format_arrow_type_as_sql`
and passes `field.is_nullable()` through. In `dbt-tasks-sa`
`renderable/unit_test.rs`, `columns_to_formatted_types` forwards it too, and
`create_values` uses the result as a `CAST` target:

```sql
SELECT CAST(1 AS INT NOT NULL) AS id, ...
```

T-SQL rejects `NOT NULL` in a `CAST`. `fabric::try_format_type` appends
` NOT NULL` and `postgres::try_format_type` appends ` not null` for a
non-nullable field. ClickHouse relies on the flag here to pick
`Nullable(...)`.

## Measured

SQL Server 2022 (16.0.4295.3), with the SQL Server adapter from #15769 before
[dbt-sqlserver-next/dbt-core#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21),
which formatted types like Fabric. A unit test whose `given` is a table built
from `select 1 as id, cast('a' as varchar(10)) as name` failed with
`Incorrect syntax near the keyword 'NOT'. (156)`. The catalog reports `id` as
NOT NULL, and the metadata adapter builds the `given` schema with that
nullability. Not measured on Fabric or Postgres.

## Fix

Pass `true` from `columns_to_formatted_types` for every adapter but
ClickHouse. A `CAST` target carries no nullability.
