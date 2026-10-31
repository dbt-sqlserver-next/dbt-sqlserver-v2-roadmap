# 00 — Current State Audit

Audited against local checkouts on 2026-10-31:

- `dbt-core/` @ `bad853106461c975757a8d2510c63cdb29eb4c66` (`sqlserver-v2-port`), merging port head `4d70a106` with upstream `main` `b601ffe4`.
- `dbt-sqlserver/` @ `2875db81103d9df6aaba4f2d1503fe2b69897f44` (`chore/1.12.2rc1`, `v1.12.1-10-g`). This checkout was not changed by the V2 refresh.
- Live verification server: SQL Server 2022 `16.0.4295.3`, local test instance.

## V2 port status

The local port is implemented as one compiled adapter slice; it is not a standalone PyPI adapter.

| Surface | Current implementation |
|---|---|
| Adapter identity | `AdapterType::SqlServer`, quoting, relation factory and SQL dialect arms are implemented. |
| Profile and init | `SqlServerDbConfig`, headless and interactive profile setup support SQL login and the implemented Entra flows. |
| Connectivity | `Backend::SQLServer` uses ADBC; current `MSSQLSERVER_DRIVER_VERSION` is `1.7.1`. |
| Runtime/catalog | SQL Server relation lookup, catalog introspection, type mapping, column construction, adapter behavior and macro package are present. |
| SQL Server macros | Vendored `dbt-sqlserver` macros include selected fixes beyond the original v1 1.12.0 sync: IDENTITY-safe empty table creation, aliased delete+insert, nullability-preserving type changes, quoted snapshot columns, and non-executing CTE schema discovery. |
| Unit-test CTEs | Model CTEs are spliced into the unit-test query where SQL Server cannot nest a `WITH`; the non-leading-`WITH` ephemeral wrapper has an alias. |

The adapter remains behind `DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true`. Local changes have not been pushed. Live PR [dbt-labs/dbt#15769](https://github.com/dbt-labs/dbt/pull/15769) must be refreshed separately; it still points at the old remote branch head until then.

## Verification

- `cargo check` for the adapter, profile, init, loader, Jinja, task and CLI crates passed.
- `cargo clippy` for those crates and all targets passed with warnings denied.
- Relevant Rust tests passed: adapter `1588/1588` (with Arrow IPC LZ4 enabled and one test thread), loader `258/258`, tasks `285/285`, profile schemas `3/3`, init `7/7`, Jinja utilities `161/161`, adapter core `20/20` plus one ignored, SQL adapter `46/46`, auth `353/353` plus three ignored, schemas `708/708` plus one ignored.
- On SQL Server 2022, the rebuilt V2 CLI connected successfully. Live dbt runs verified full-refresh from an IDENTITY source, composite-key delete+insert, varchar widening with `NOT NULL` and data preserved, a four-step non-null type rewrite, snapshot migration of a reserved-word column, non-leading-WITH ephemeral references, and semicolon-prefixed CTE schema discovery without evaluating a divide-by-zero query.
- All scratch objects were dropped after testing.

## Remaining gaps

- Entra authentication is unit-tested but lacks live-server verification; the live run used SQL authentication. Minimum catalog permissions have not been established for least-privilege users.
- Windows/integrated auth, named instances, case-sensitive collation behavior, dynamic data masking/denies, custom indexes and several v1-only materializations remain deferred or unsupported; see `05-open-questions-and-risks.md` and the upstream PR draft.
- `dbt.date_spine` and `dbt_utils.expression_is_true` remain known SQL Server package-compatibility gaps.
- The default parallel `dbt-adapter` test run showed a Databricks catalogs-v2 shared-state failure and an Arrow IPC LZ4 feature failure. Serial execution with the LZ4 feature passed the full adapter suite.
