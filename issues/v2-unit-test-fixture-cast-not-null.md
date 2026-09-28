---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: https://github.com/dbt-sqlserver-next/dbt-core/pull/21
---

# [v2 Bug] Unit-test fixtures cast to `<type> NOT NULL` when the `given` relation's column is NOT NULL

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

`1330d0219` added a `nullable` parameter to `TypeOps::format_arrow_type_as_sql` and passes `field.is_nullable()` through. In `dbt-tasks-sa` `renderable/unit_test.rs`, `columns_to_formatted_types` forwards it too, and `create_values` uses the result as a `CAST` target:

```sql
SELECT CAST(1 AS INT NOT NULL) AS id, ...
```

T-SQL rejects `NOT NULL` in a `CAST`. `fabric::try_format_type` appends ` NOT NULL` and `postgres::try_format_type` appends ` not null` for a non-nullable field. ClickHouse relies on the flag here to pick `Nullable(...)`.

### Expected Behavior

Fixture values are cast to the column's type without nullability, and the unit test runs.

### Steps To Reproduce

1. Build the SQL Server adapter from #15769 as it was before [dbt-sqlserver-next/dbt-core#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21), when it formatted types like Fabric.
2. Make the `given` relation a table built from `select 1 as id, cast('a' as varchar(10)) as name`. The catalog reports `id` as NOT NULL, and the metadata adapter builds the `given` schema with that nullability.
3. Run a unit test on a model that reads it.

### Relevant log output

```shell
Incorrect syntax near the keyword 'NOT'. (156)
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source with the SQL Server adapter from #15769; SQL Server 2022 (16.0.4295.3)

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: SQL Server (#15769). Not measured on Fabric or Postgres.

In 1.x, the same `given` table (`id` NOT NULL in `sys.columns`) passes: dbt-core 1.12.3, dbt-sqlserver 1.12.0, same server.

Fix: pass `true` from `columns_to_formatted_types` for every adapter but ClickHouse. A `CAST` target carries no nullability.
