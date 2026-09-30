---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# Seeds overflow `int` and `varchar(8000)`

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

`convert_number_type` returns `int` for a column without decimals, and `convert_text_type` returns `varchar(<longest value in bytes>)`.

| Seed | Result |
|---|---|
| `amt` = `5000000000` | `The conversion of the varchar value '5000000000' overflowed an int column.` |
| `t` = 9,000 `x` characters | `The size (9000) given to the column 't' exceeds the maximum allowed for any data type (8000).` |

With `column_types`, `bigint` loads `9223372036854775807` and `-3000000000`, and `numeric(38,0)` loads `9223372036854775808`. `int` stays for columns within `-2147483648..2147483647` (both ends load today), so existing seeds keep their type. `convert_number_type` can check the column's minimum and maximum and return `int`, `bigint` (within 64 bits) or `numeric(38,0)`; `convert_text_type` can return `varchar(max)` above 8,000 bytes. A column with decimals stays `float`.

## Steps to reproduce

```
# seeds/big_int.csv
id,amt
1,5000000000
```

```
# seeds/long_txt.csv: t is 9000 characters
id,t
1,xxxx…
```

`dbt seed` fails on both.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
