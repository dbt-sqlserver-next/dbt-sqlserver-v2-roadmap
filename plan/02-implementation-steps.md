# 02 — V2 Port Implementation Status

Current as of 2026-10-31. The SQL Server port is implemented locally in
`dbt-core:sqlserver-v2-port` at merge commit `bad853106`, based on upstream
`main` `b601ffe4`. The local branch has not been pushed; upstream PR #15769
still points at remote head `4d70a106`.

## Implemented

- `AdapterType::SqlServer`, SQL dialect behavior, profile schema, authentication,
  `dbt init`, relation/type/catalog support and SQL Server adapter match arms.
- Vendored `dbt-sqlserver` macro package.
- MSSQL ADBC driver `1.7.1` from current upstream.
- Macro fixes for IDENTITY-safe empty full refresh, delete+insert target aliasing,
  `NOT NULL` preservation during widening, and quoted snapshot columns.
- CTE schema discovery through `sp_describe_first_result_set`, an empty typed
  result for CTE schema probes, leading `;WITH` recognition, view-body leading
  terminator removal, and SQL Server ephemeral wrapper aliases.
- Unit-test model CTE lifting for T-SQL.

## Verification

- `cargo check` for the changed adapter, profile, loader, task and CLI crates;
  `cargo build -p dbt-sa-cli`; formatting and clippy (`-D warnings`) pass.
- Rust tests: `dbt-adapter --lib` 1588 passed with Arrow IPC LZ4 enabled and one
  test thread; `dbt-loader` 258; `dbt-tasks-sa` 285; `dbt-profile-schemas` 3;
  `dbt-init` 7; `dbt-jinja-utils` 161; `dbt-adapter-core` 20; `dbt-adapter-sql`
  46; `dbt-auth` 353; `dbt-schemas` 708. Ignored tests are not counted as passes.
- Live SQL Server 2022 `16.0.4295.3` with SQL auth: V2 CLI `debug` and model
  runs pass; full refresh preserves identity values without an identity target;
  composite-key delete+insert preserves nonmatching rows; varchar widening and
  the four-step rewrite preserve `NOT NULL` and values; snapshot CTEs add a
  reserved-word column; non-leading-WITH ephemeral refs build; semicolon-prefixed
  CTE schema probing returns metadata and zero rows without evaluating a
  divide-by-zero expression. Scratch objects were dropped.

## Remaining before an RC

- Re-run upstream CI after the local branch is published and the public-fork CI
  approval is granted. No push or live PR edit has been made.
- Live Entra validation and minimum catalog-permission requirements remain open.
- Preserve known package limitations: `dbt.date_spine`,
  `dbt_utils.expression_is_true`, Windows/integrated auth, named instances,
  collation-aware case folding, dynamic data masking/denies and custom indexes.
- The view-plan-reuse optimization in v1 `916fc235` remains an optional follow-up;
  it is not required for this refresh.
