---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: v1-seed-column-types.md
---

# The `mssql` extra allows mssql-python versions that fail seeds with 16-digit decimals

Measured on SQL Server 2022 (16.0.4295.3, Linux), against dbt-sqlserver v1.12.0.

`pyproject.toml` requires `mssql-python>=1.7.1` for the `mssql` extra and the `dev` group, and `uv.lock` resolves 1.14.0. Every mssql-python release from 1.7.1 to 1.14.0 fails a parameterized insert where a `Decimal` with 16 or more significant digits follows a non-decimal parameter. A seed does exactly this: `insert into t (id, d) values (?, ?), (?, ?)`.

```python
cur.execute("create table dbo.t (id int, d float)")
cur.execute("insert into dbo.t (id, d) values (?, ?), (?, ?)",
            [1, Decimal("0.1"), 2, Decimal("1234567890123456.5")])
# Driver Error: Numeric value out of range; DDBC Error: [Microsoft]Numeric value out of range
```

| mssql-python | 15 digits | 16 digits | 17 digits |
|---|---|---|---|
| 1.7.1 – 1.14.0 | ok | error | error |
| 1.15.0 | ok | ok | ok |

The error carries no SQL Server text, so it comes from the driver. It also occurs with a `numeric(38,2)` column and with `float` or `str` in place of the `int`, and `pyodbc` runs the same insert. A `Decimal` first, or on its own, works on every version.

Raising the floor to `mssql-python>=1.15.0` in both places removes the failure. It excludes installs on 1.7.1 through 1.14.0 that have no such seed, so the alternative is a README note that seeds with 16-digit decimals need 1.15.0 or newer.

## Steps to reproduce

```
# seeds/dec.csv
id,d
1,0.1
2,1234567890123456.5
```

`dbt seed` with `backend: mssql-python` on 1.14.0 fails; on 1.15.0 it loads.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python 1.7.1 to 1.15.0
- dbt-sqlserver: v1.12.0
