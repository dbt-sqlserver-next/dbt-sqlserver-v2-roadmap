---
target_repo: dbt-labs/dbt-adapters
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `dbt.date_spine`/`generate_series` nests a `WITH` block inside another CTE's body, which T-SQL (SQL Server, Fabric) rejects

Measured against SQL Server 2022 (16.0.4265.3, Linux), but Fabric Warehouse
runs the same T-SQL engine and shares this macro unmodified — not specific to
on-prem SQL Server.

## Summary

`macros/utils/date_spine.sql`'s `default__date_spine`:

```jinja
with rawdata as (
    {{ dbt.generate_series(dbt.get_intervals_between(start_date, end_date, datepart)) }}
),
all_periods as (...),
filtered as (...)
select * from filtered
```

embeds `generate_series`'s return value — itself a full
`with p as (...), unioned as (...) select ... from unioned where ... order by ...`
statement — directly inside the `rawdata` CTE's parentheses. Reproduced
directly against a live SQL Server container:

```sql
with rawdata as (
    with p as (select 1 as n)
    select n from p
)
select * from rawdata
```

fails with:

```
Msg 156: Incorrect syntax near the keyword 'with'.
```

T-SQL only allows a single `WITH` introducing a flat, comma-separated CTE
list at the start of a statement/batch — a CTE's own body cannot contain
another `WITH`-introduced list.

A second, independent T-SQL restriction sits in `default__date_spine` itself:
its `all_periods` CTE computes each period as
`dateadd(..., row_number() over (order by 1) - 1, ...)`, and an ordinal
literal in a window's `ORDER BY` is rejected —
`Msg 5308: Windowed functions ... do not support integer indices as ORDER BY
clause expressions.` — so flattening the CTE alone isn't sufficient; both
need addressing together.

A third restriction is in `default__generate_series`, but hasn't shown up in a
run yet because the parse fails on the nested `WITH` first: the macro ends in
`order by generated_number`, and T-SQL rejects an `ORDER BY` inside a CTE
unless `TOP`, `OFFSET` or `FOR XML` is also specified (Msg 1033).

## Impact

Breaks `metricflow_time_spine` (and anything else using `dbt.date_spine`)
for SQL Server and, since the macro is unmodified between the two, almost
certainly Fabric as well. No `fabric__`/`sqlserver__` override for
`date_spine`/`generate_series` exists today. Discovered while smoke-testing
the SQL Server Fusion (v2) adapter against jaffle-shop
(`dbt-sqlserver-next/dbt-core` Part 10) — this is a pre-existing gap in the
shared macro, not a v2-only regression, though not yet confirmed against a
live Fabric warehouse.

## Suggested fix

A `sqlserver__date_spine` override (and ideally `fabric__date_spine`), since
two of the three problems are in `date_spine`'s own body and an override of
`generate_series` alone can't reach them. It:
- builds the number series in the same top-level `WITH` list as its own CTEs,
  either inline or via a `generate_series` variant that returns CTE fragments
  rather than a self-contained statement, so no second `WITH` is nested;
- drops the trailing `order by generated_number` from the nested series; and
- replaces the ordinal `order by 1` with `order by (select null)`, which
  T-SQL accepts as an explicitly-unordered window.

Needs a working end-to-end test against a live instance before landing —
this issue documents root cause, not a verified fix.
