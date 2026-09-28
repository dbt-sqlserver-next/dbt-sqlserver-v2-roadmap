---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: v2-execute-inner-last-statement-clobbers-fetch-result.md
---

# [v2 Bug] Fabric returns the last statement's result from a multi-statement `statement()` block; 1.x returned the first

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

`execute_inner` (`dbt-adapter` `adapter_impl.rs`) splits a `statement()` block into physical statements for every adapter except BigQuery, DuckDB, LakeCompute and SQL Server, runs them in turn, and returns the last one's result. Fabric is split.

dbt-fabric 1.x sends the whole block as one batch through mssql-python, and a T-SQL driver positions the cursor on the first result. So a `fetch_result=True` block that ends in a cleanup statement returns its `select` in 1.x and an empty table in v2:

```jinja
{% call statement('probe', fetch_result=True) %}
  EXEC('create view dbo.v as select 1 as n');
  select count(*) as failures from dbo.v;
  EXEC('drop view dbo.v')
{% endcall %}
```

Fabric's own v2 macros don't hit this: `fabric__get_test_sql` and `fabric__get_unit_test_sql` are single statements. A user or package macro written against dbt-fabric 1.x can.

### Expected Behavior

A multi-statement block on Fabric returns the result 1.x returned.

### Steps To Reproduce

Not run on Fabric; no warehouse was available. The driver behavior, measured with plain `cursor.execute()` on SQL Server 2022:

| SQL | pyodbc 5.3.0, mssql-python | psycopg2 2.9.13 | psycopg 3.3.6 |
|---|---|---|---|
| `create view; select; drop view` | the `select` | nothing | nothing |
| `select 1 as a; select 2 as b` | `a` | `b` | `a` |

v2's "last result" matches psycopg2, which dbt-postgres 1.x pins (`psycopg2-binary<3.0`). It does not match the T-SQL drivers.

### Relevant log output

_No response_

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source; drivers measured on SQL Server 2022 (16.0.4295.3) and PostgreSQL 16.15

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [X] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: Fabric. dbt-fabric 1.11.1 depends on `mssql-python>=1.4.0`.

SQL Server had the same gap and now sends each block whole (#15769): T-SQL scopes a `DECLARE`'d variable to its batch and `MERGE` keeps its terminator, so splitting also broke those. Adding Fabric to the same arm would restore the 1.x result and keep the change out of other adapters. Whether the Fabric ADBC driver returns the first result set of a batch is not measured.
