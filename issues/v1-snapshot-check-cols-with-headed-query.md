---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: https://github.com/dbt-msft/dbt-sqlserver/pull/864
---

# A `check` snapshot with a `check_cols` list fails on its second run when its SQL starts with `WITH`

A `check` snapshot with `check_cols: [...]` fails on every run after the first
with `Incorrect syntax near the keyword 'with'. (156)` when its SQL starts with
`WITH`. That covers a snapshot with its own CTE, and every snapshot that
selects from an ephemeral model, since dbt prepends ephemerals as CTEs (#864
enables the ephemeral tests).

```sql
{% snapshot snap_cte %}
{{ config(unique_key='id', strategy='check', check_cols=['name'], target_schema=target.schema) }}
with s as (select * from {{ ref('src') }}) select * from s
{% endsnapshot %}
```

The first run passes. The second fails on this query:

```sql
select TOP 0 * from (
    select name from (
        with s as (...) select * from s
    ) subq
) as __dbt_sbq where 0 = 1
```

Measured on SQL Server 2022 (16.0.4295.3), master `d5b32c6d` plus #864, with
dbt-core 1.12.3 and dbt-adapters 1.24.5. `check_cols: all`, `timestamp`
snapshots and a `check_cols` list over SQL without a `WITH` all pass.

## Cause

dbt-adapters'
[`snapshot_check_all_get_existing_columns`](https://github.com/dbt-labs/dbt-adapters/blob/0514bae93e4e216256ed49f6fb4024e43f295b57/dbt-adapters/src/dbt/include/global_project/macros/materializations/snapshots/strategies.sql#L111-L120)
wraps the snapshot SQL as `select <check_cols> from ( <sql> ) subq` to read
the columns' casing. T-SQL can't nest a `WITH` in a subquery. The `all`
branch passes the SQL through unwrapped, so it works.

The macro isn't dispatched, but a macro of the same name in the adapter
package wins over dbt's global project
([`macro_resolver.py`](https://github.com/dbt-labs/dbt-core/blob/v1.12.3/core/dbt/context/macro_resolver.py#L44-L51)).

## Options

1. **Override the macro in the adapter (recommended).** Read the query's
   columns unwrapped, as the `all` branch does, and pick the `check_cols` from
   them by case-insensitive name, raising a compile error for a name that
   isn't there. Prototyped as a root-project macro. It passes both snapshots
   on repeated runs and closes the old version of a changed row. `NAME`
   matches `name`, and `nope` fails with
   `check_cols column 'nope' is not in the snapshot query`. The cost is a
   copy of an upstream macro to keep in sync. The match is case-insensitive
   even under a case-sensitive collation, where upstream would fail on a
   miscased name.
2. **Fix it upstream.** Every other adapter accepts the subquery, so this
   would be a T-SQL-only change in dbt-adapters.
3. **Document it.** The workaround is `check_cols: all`, or a `WITH`-free
   snapshot query.
