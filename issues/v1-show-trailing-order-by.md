---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `dbt show` fails when the query ends in `order by`

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

`sqlserver__get_limit_sql` treats a query as ordered only when its last line starts with `order by`. Otherwise it appends `order by (select null) offset 0 rows fetch first n rows only`, which fails with `Incorrect syntax near the keyword 'order'` on both of these:

```
select a from {{ ref('ord') }} order by a

select a from {{ ref('ord') }}
order by
  a
```

Reading the text after the last closing parenthesis for `order by` covers both.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
