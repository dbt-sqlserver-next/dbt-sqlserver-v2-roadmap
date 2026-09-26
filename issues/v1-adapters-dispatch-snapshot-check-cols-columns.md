---
target_repo: dbt-labs/dbt-adapters
type: feature
status: draft
related: https://github.com/dbt-msft/dbt-sqlserver/issues/865
---

# Dispatch how the `check` snapshot strategy reads the `check_cols` columns

`snapshot_check_all_get_existing_columns` isn't dispatched, and for a
`check_cols` list it reads the columns' casing from

```sql
select <check_cols> from ( <snapshot sql> ) subq
```

That subquery isn't valid SQL on every database. T-SQL can't nest a `WITH` in
one, so on SQL Server a `check` snapshot with a `check_cols` list fails from its
second run whenever its SQL starts with `WITH`, and every snapshot that selects
from an ephemeral model does
([dbt-sqlserver#865](https://github.com/dbt-msft/dbt-sqlserver/issues/865)).

With no dispatch, an adapter's only fix is to copy the whole macro under the
same name, since an adapter package's macro wins over the global project's.
Oracle, Teradata, StarRocks, SAP HANA Cloud, Doris and MySQL adapters already
do, several with the older two-argument signature. dbt-sqlserver is about to
([dbt-sqlserver#867](https://github.com/dbt-msft/dbt-sqlserver/pull/867)). Each copy drifts from upstream.

## Proposal

Dispatch only the step that differs: reading the query's columns. The rest of
the macro stays as it is.

```jinja
{% macro get_snapshot_check_cols_query_columns(node, check_cols_config) -%}
  {{ return(adapter.dispatch('get_snapshot_check_cols_query_columns', 'dbt')(node, check_cols_config)) }}
{%- endmacro %}

{% macro default__get_snapshot_check_cols_query_columns(node, check_cols_config) -%}
  {# today's if / elif / else, moved unchanged #}
  {{ return(query_columns) }}
{%- endmacro %}
```

`snapshot_check_all_get_existing_columns` then calls
`get_snapshot_check_cols_query_columns(node, check_cols_config)` in place of
the `if` block.

## Why a new name, not dispatching the existing one

Existing adapters see no change:

- An adapter or root project that shadows `snapshot_check_all_get_existing_columns`
  keeps winning over the global macro, so it never reaches the new dispatch.
- Every other adapter resolves to `default__`, which is today's code.
- The new name has no matches in GitHub code search, so no existing
  `<adapter>__get_snapshot_check_cols_query_columns` gets picked up by accident.

Dispatching `snapshot_check_all_get_existing_columns` itself isn't safe.
dbt-exasol defines `exasol__snapshot_check_all_get_existing_columns(node, target_exists)`,
which nothing calls today because Exasol uses the global
`snapshot_check_strategy`. A dispatch would start calling it with three
arguments, and Jinja raises `macro ... takes not more than 2 argument(s)`. It
also reads `node['injected_sql']`, which dbt no longer sets.

Checked against dbt-adapters `main` (`0514bae9`), which matches 1.24.5, and
dbt-exasol `4dadcc43`. Code search covers default branches only.
