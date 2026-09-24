---
target_repo: dbt-labs/dbt-utils
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `expression_is_true` selects an unaliased `1`, which T-SQL rejects inside the test wrapper

`default__test_expression_is_true` (`macros/generic_tests/expression_is_true.sql`):

```jinja
{% set column_list = '*' if should_store_failures() else "1" %}
```

Without `store_failures` (the default) the test query is `select 1 from ...`.
dbt wraps every test query before counting failures, and T-SQL requires every
column of a view, derived table or CTE to be named.

## Measured

SQL Server 2022 (16.0.4295.3, Linux), dbt-core 1.12.3, dbt-sqlserver 1.12.0rc4,
dbt_utils 1.4.1. dbt-sqlserver wraps the test query in a view:

| | Result |
|---|---|
| `expression_is_true` test | `Create View or Function failed because no column name was specified for column 1. (4511)` |
| same, `--store-failures` | PASS |
| with `"1 as expression_is_true"` | PASS; a false expression reports `FAIL 3` |

The same query in a derived table or CTE, as dbt-adapters' and dbt-fabric's
`get_test_sql` wrap it, is Msg 8155 (`No column name was specified for column 1
of 'dbt_internal_test'`).

`expression_is_true` is the only generic test in the package that selects a bare
literal.

## Why a T-SQL package doesn't cover it

dbt-msft/tsql-utils overrides this macro as `fabric__test_expression_is_true`
(`select 1 as col`). All of its overrides use the `fabric__` prefix, and
dbt-sqlserver stopped inheriting from dbt-fabric in 1.9.1, so dispatch for the
`sqlserver` adapter goes `sqlserver__` → `default__` and never reaches them.

## Fix

Alias the literal in the default, which is valid on every adapter:

```jinja
{% set column_list = '*' if should_store_failures() else "1 as expression_is_true" %}
```
