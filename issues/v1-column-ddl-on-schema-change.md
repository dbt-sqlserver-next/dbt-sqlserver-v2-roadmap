---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/881
related: ../plan/04-testing-and-validation.md
---

# Widening a column drops `NOT NULL`, and a snapshot can't add a column that needs quoting

Two bugs in the code that changes the columns of an existing incremental or snapshot table.

`sqlserver__alter_column_type` emits `alter column "c" <new type>` with no nullability, which makes the column nullable. The four-step rewrite (`prefer_single_alter_column: false`) adds the new column as nullable, with the same result. The model keeps running; the table silently loses the `NOT NULL` the first time a wider value arrives. Snapshots expand columns through the same macro.

`sqlserver__create_columns` (snapshots) renders column names unquoted, so a snapshot whose query gains a column named `order`, or one with a space, fails with `Incorrect syntax near the keyword 'order'.`

## Steps to reproduce

Widening:

```sql
-- models/m.sql
{{ config(materialized='incremental', as_columnstore=false) }}
select isnull(cast('keep me' as varchar(35)), '') as c, 1 as id
```

`dbt run` creates `c varchar(35) not null`. Change `varchar(35)` to `varchar(63)` and `dbt run` again: `c` is `varchar(63)` and nullable.

```sql
select max_length, is_nullable from sys.columns where object_id = object_id('<schema>.m') and name = 'c';
-- 63, 1
```

Snapshot column:

```sql
-- snapshots/snap.sql
{% snapshot snap %}
{{ config(unique_key='id', strategy='check', check_cols='all', target_schema=target.schema) }}
select 1 as id{% if var('v2', false) %}, 'x' as [order]{% endif %}
{% endsnapshot %}
```

`dbt snapshot`, then `dbt snapshot --vars '{v2: true}'`.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python 1.14.0 and pyodbc (ODBC Driver 18)
- Authentication, ODBC driver, OS: SQL login, ODBC Driver 18, Linux
- Database collation: SQL_Latin1_General_CP1_CS_AS
- Other dbt packages: none

`dbt --version`:
```text
Core: 1.12.3
dbt-sqlserver: 1.12.0
```

## Log excerpt
```text
Database Error in snapshot snap (snapshots/snap.sql)
  Driver Error: Syntax error or access violation; DDBC Error: [Microsoft][SQL Server]Incorrect syntax near the keyword 'order'.
```

The widening raises no error.
