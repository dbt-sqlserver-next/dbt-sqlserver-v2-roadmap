---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# The README doesn't state several limits of the utils macros, snapshots and data tests

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

Behaviour that follows from SQL Server and can't change without altering existing results, so it needs a README statement instead of a code change.

- **`hash`** converts the input to `varchar(8000)` first, so strings that share their first 8000 characters hash equally (9,000 and 9,500 `x` characters give the same MD5), and non-code-page characters become `?`. Changing it alters every existing hash value.
- **`split_part`** returns `varchar`, so a non-code-page character in a part becomes `?` (`N'日本'` returns `??`). Returning `nvarchar` changes the type of every column built from it.
- **`listagg`** uses `string_agg` on the measure as typed; a `varchar(n)` measure fails with `STRING_AGG aggregation result exceeded the limit of 8000 bytes` once the result passes 8,000 bytes. Casting to `varchar(max)` changes the column type, so the README states the limit and the cast to use.
- **`snapshot_hash_arguments`** converts each argument to `varchar(8000)` before hashing, so two values that differ only in characters outside the code page hash equally: `N'日本'` and `N'本日'` both give `EA03FCB8C47822BCE772CF6C07D0EBBB`. `dbt_scd_id` is the join key for existing snapshot rows, so a different hash would orphan them.
- **Data tests** run inside `sqlserver__get_test_sql`, which wraps the query in a view, so a test whose query isn't valid as a view body fails. A query ending in `ORDER BY` fails with `The ORDER BY clause is invalid in views, inline functions, derived tables, subqueries, and common table expressions, unless TOP, OFFSET or FOR XML is also specified.` A query that returns two columns with the same name, such as `select * from a join b on …`, fails at `create view`. Queries that start with a CTE work, which the view is there for.
- **`length` and `listagg`**, if [v1-utils-macros-edge-cases](v1-utils-macros-edge-cases.md) is taken: `length` takes a string expression, and `listagg` doesn't support `limit_num`.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
