---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/883
related: v1-mssql-python-minimum-version.md
---

# Seeds overflow `int` and `varchar(8000)`

A seed column of whole numbers is always created as `int`, and a text column as `varchar(<longest value in bytes>)`. A value beyond `int`, or text longer than 8,000 bytes, fails the seed:

| Seed | Error |
|---|---|
| `n` = `5000000000` | `The conversion of the varchar value '5000000000' overflowed an int column.` |
| `n` = `9223372036854775808` | `Arithmetic overflow error converting expression to data type int.` |
| `t` = 9,000 characters | `The size (9000) given to the column 't' exceeds the maximum allowed for any data type (8000).` |

`bigint` for whole numbers beyond `int`, `numeric(38,0)` beyond `bigint`, and `varchar(max)` above 8,000 bytes load all three. Columns that load today would keep their type.

## Steps to reproduce

```text
# seeds/bigints.csv
id,n
1,5000000000
```

```text
# seeds/long_text.csv: t is 9000 characters
id,t
1,xxxx…
```

`dbt seed`.

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
Database Error in seed bigints (seeds/bigints.csv)
  Driver Error: Numeric value out of range; DDBC Error: [Microsoft][SQL Server]The conversion of the varchar value '5000000000' overflowed an int column.
```
