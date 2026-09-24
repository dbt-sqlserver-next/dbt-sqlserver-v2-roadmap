---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `dbt.date_spine` fails on SQL Server: the default nests `generate_series`'s `WITH` inside a CTE

dbt-sqlserver has no `sqlserver__date_spine` or `sqlserver__generate_series`,
so it uses dbt-adapters' defaults. `default__date_spine` puts a full
`generate_series` statement inside a CTE:

```jinja
with rawdata as (
    {{ dbt.generate_series(...) }}   {# with p as (...), unioned as (...) select ... order by generated_number #}
),
all_periods as (
    select {{ dbt.dateadd(datepart, "row_number() over (order by 1) - 1", start_date) }} ...
```

T-SQL rejects three things here:

| Construct | Error |
|---|---|
| `WITH` inside a CTE body | Msg 156, `Incorrect syntax near the keyword 'with'` |
| `order by generated_number` inside a CTE | Msg 1033 |
| `row_number() over (order by 1)` | Msg 5308 |

## Measured

SQL Server 2022 (16.0.4295.3, Linux), dbt-core 1.12.3, dbt-sqlserver 1.12.0rc4.
A model containing only
`{{ dbt.date_spine("day", "cast('2024-01-01' as date)", "cast('2024-01-11' as date)") }}`:

- as is: `Incorrect syntax near the keyword 'with'. (156)`
- with the override below: builds 10 rows, 2024-01-01 through 2024-01-10, the
  same end-exclusive range as the default

This breaks `metricflow_time_spine` and anything else built on `date_spine`.
The v2 adapter's macro package has the same gap.

## Fix

Add `sqlserver__date_spine`, which puts the series in the same `WITH` list and
uses `generated_number` directly instead of `row_number()`:

```jinja
{% macro sqlserver__date_spine(datepart, start_date, end_date) %}
    {%- set upper_bound = dbt.get_intervals_between(start_date, end_date, datepart) -%}
    {%- set n = dbt.get_powers_of_two(upper_bound) -%}
    with p as (
        select 0 as generated_number union all select 1
    ),
    unioned as (
        select
        {% for i in range(n) %}
        p{{ i }}.generated_number * power(2, {{ i }}){% if not loop.last %} + {% endif %}
        {% endfor %}
        + 1 as generated_number
        from
        {% for i in range(n) %}
        p as p{{ i }}{% if not loop.last %} cross join {% endif %}
        {% endfor %}
    ),
    all_periods as (
        select {{ dbt.dateadd(datepart, "generated_number - 1", start_date) }} as date_{{ datepart }}
        from unioned
        where generated_number <= {{ upper_bound }}
    ),
    filtered as (
        select *
        from all_periods
        where date_{{ datepart }} <= {{ end_date }}
    )
    select * from filtered
{% endmacro %}
```

`sqlserver__generate_series` alone can't fix it: two of the three errors are in
`date_spine`'s own body.
