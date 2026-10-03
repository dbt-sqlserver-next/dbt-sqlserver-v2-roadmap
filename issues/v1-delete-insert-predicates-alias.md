---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/882
related: ../plan/04-testing-and-validation.md
---

# `delete+insert` predicates can't use `DBT_INTERNAL_DEST`

dbt documents `DBT_INTERNAL_DEST` as the alias an `incremental_predicates` entry uses for the target. `sqlserver__get_delete_insert_merge_sql` deletes from the bare target, so the alias doesn't exist and the second run fails with `The multi-part identifier "DBT_INTERNAL_DEST.id" could not be bound.` `delete DBT_INTERNAL_DEST from <target> as DBT_INTERNAL_DEST` defines it, as dbt-adapters' default macro does.

## Steps to reproduce

```sql
-- models/m.sql
{{ config(materialized='incremental', incremental_strategy='delete+insert', unique_key='id',
          incremental_predicates=['DBT_INTERNAL_DEST.id > 1']) }}
select id from (values (1), (2)) as t(id)
```

`dbt run` twice. A list `unique_key` fails the same way.

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
Database Error in model m (models/m.sql)
  Driver Error: Syntax error or access violation; DDBC Error: [Microsoft][SQL Server]The multi-part identifier "DBT_INTERNAL_DEST.id" could not be bound.
```
