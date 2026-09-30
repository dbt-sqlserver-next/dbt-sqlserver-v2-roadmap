---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# Widening a column drops `NOT NULL`, a snapshot can't add a column named like a keyword, and the type-expansion row count runs unconditionally

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

Three findings on the code that changes a column of a table that already exists.

## `sqlserver__alter_column_type` makes the column nullable

`sqlserver__alter_column_type` emits `alter table … alter column "c" <new type>` with no nullability. SQL Server then makes the column nullable:

```sql
select isnull(cast('a' as varchar(5)), '') as c into dbo.qw_t;   -- c is NOT NULL
alter table dbo.qw_t alter column [c] varchar(10);
select name, is_nullable from sys.columns where object_id = object_id('dbo.qw_t');
-- c, 1
```

A same-family widening (`varchar(5)` to `varchar(10)`) takes this path by default, so an incremental or snapshot table whose column was `NOT NULL` (for example from an `isnull(...)` expression) loses the constraint the first time a wider value arrives. The four-step path (`prefer_single_alter_column: false`) has the same effect: on a `NOT NULL` `varchar(5)` it leaves `c varchar(10)` nullable, and `c` moves after the other columns. On a column with a default constraint it fails with `The object 'df_c' is dependent on column 'c'.` and leaves `c__dbt_alter` behind. Appending `not null` when `sys.columns.is_nullable` is 0 keeps the constraint on the single-statement path.

## `create_columns` doesn't quote column names

`sqlserver__create_columns` (snapshots) renders `{{ column_entry.name }} {{ column_entry.data_type }}` bare. A snapshot whose query gains a column named `order` fails on the run that adds it:

```sql
{% snapshot snap %}
{{ config(target_schema='s', unique_key='id', strategy='check', check_cols='all') }}
select 1 as id, 'a' as v {% if var('v2', false) %}, 'x' as [order] {% endif %}
{% endsnapshot %}
```

`dbt snapshot`, then `dbt snapshot --vars '{v2: true}'` fails with `Incorrect syntax near the keyword 'order'.` Names containing spaces or other characters that need quoting fail the same way. `adapter.quote(column_entry.name)` matches `sqlserver__alter_relation_add_remove_columns`.

## `expand_column_types` counts rows before it knows it needs to

With `dbt_sqlserver_enable_safe_type_expansion` on and a positive `column_type_expansion_max_rows` (default 1,000,000), `_safe_expansion_allowed` runs `SELECT COUNT_BIG(*)` on the target on every incremental and snapshot run, including runs where no column differs.

| Target | `COUNT_BIG(*)` |
|---|---|
| 20M-row heap | 0.15 s |
| 20M-row clustered columnstore | 0.003 s |

Counting only when a column qualifies for a cross-family promotion skips it on every run that has none. The gain is small: it applies to opt-in users with rowstore or heap targets.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
