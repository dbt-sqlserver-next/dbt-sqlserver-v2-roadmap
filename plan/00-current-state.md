# 00 — Current State Audit

Audited against local checkouts on 2026-11-24:

- `dbt-core/` @ `51db264c1` (`sqlserver-v2-port`), based on upstream `main`
  `aa8534461`.
- `dbt-sqlserver/` @ `1b45b33c` (`master`, `v1.12.3`), clean.
- Live verification server: SQL Server 2022 `16.0.4295.3`, collation
  `SQL_Latin1_General_CP1_CS_AS`, local test instance.

## V2 port status

The local port is implemented as one compiled adapter slice; it is not a standalone PyPI adapter.

| Surface | Current implementation |
|---|---|
| Adapter identity | `AdapterType::SqlServer`, quoting, relation factory and SQL dialect arms are implemented. |
| Profile and init | `SqlServerDbConfig`, headless and interactive profile setup support SQL login and the implemented Entra flows; `schema_authorization` sets the owner of a schema it creates. |
| Connectivity | `Backend::SQLServer` uses ADBC; current `MSSQLSERVER_DRIVER_VERSION` is `1.7.1`. |
| Runtime/catalog | SQL Server relation lookup, catalog introspection, type mapping, column construction, adapter behavior and macro package are present. |
| SQL Server macros | Vendored `dbt-sqlserver` macros are synced with v1 1.12.3: IDENTITY-safe empty table creation, aliased delete+insert, nullability-preserving type changes, quoted snapshot columns, non-executing CTE schema discovery, sub-day and week `date_trunc`, plan-reuse through `sp_executesql` in the drop and view-render paths, and the backup-relation drop guard. |
| Unit-test CTEs | Model CTEs are spliced into the unit-test query where SQL Server cannot nest a `WITH`; the non-leading-`WITH` ephemeral wrapper has an alias. |
| Identifier escaping | Relation components render through `escaped_identifier`, which doubles the T-SQL delimiter; a relation or schema name containing `"` renders as `"a""b"`. |

The adapter remains behind `DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true`. The local
`sqlserver-v2-port` branch is pushed; live PR
[dbt-labs/dbt#15769](https://github.com/dbt-labs/dbt/pull/15769) points at
`51db264c1`, the local head, and is mergeable, blocked on the required review.

## Verification

- `cargo check` passes for the adapter, adapter-core, adapter-engine,
  adapter-sql, schemas, Jinja utilities, loader, tasks, init, profile schemas
  and CLI crates.
- `cargo clippy` for the adapter, adapter-core, adapter-engine, adapter-sql,
  Jinja utilities, schemas, loader, tasks, init, profile schemas and CLI crates
  passed with warnings denied and `--all-targets`.
- Rust tests: adapter `1733/1733` (needs `--features arrow-ipc/lz4` and
  `--test-threads=1`), adapter-sql `95/95` plus two ignored, loader `66/66` and
  `337/337`, Jinja utilities `178/178`, schemas `788/788` plus one ignored, auth
  `377/377` plus four ignored, tasks `334/334`, adapter-core `20/20` plus one
  ignored, profile schemas `3/3`, init `12/12`.
- On SQL Server 2022, the rebuilt V2 CLI connected (`dbt debug`) and a cold
  `dbt build` of six models, one seed, one snapshot, five tests and one unit
  test passed `15/15`, then again on a second build.
- Live checks covered: seed columns typed `bigint`/`varchar(max)` where the data
  needs it; a `money`/`smallmoney` model under an enforced contract; `date_trunc`
  for hour, minute, second, week, day and month against expected values; a view
  rebuilt only when its body changed (`modify_date` stable across an unchanged
  run) and dropped for a table under the same name; `sync_all_columns` widening
  `varchar(5)` to `varchar(50)` with `NOT NULL` and rows preserved; snapshot
  migration of a reserved-word column; full refresh from an IDENTITY source
  without an IDENTITY target; and `"a""b"` rendering for a name containing `"`.
- All scratch objects were dropped after testing.

## Remaining gaps

- Entra authentication is unit-tested but lacks live-server verification; the live run used SQL authentication. Minimum catalog permissions have not been established for least-privilege users.
- Windows/integrated auth, named instances, case-sensitive collation folding, dynamic data masking/denies, custom indexes and several v1-only materializations remain deferred or unsupported; see `05-open-questions-and-risks.md` and the upstream PR draft.
- `dbt.date_spine` and `dbt_utils.expression_is_true` remain known SQL Server package-compatibility gaps.
- The `dbt-adapter` test suite needs `--features arrow-ipc/lz4`; without it the
  IPC round-trip test panics on LZ4 compression. The default parallel run has
  also shown shared-state failures in other adapters' tests, so the suite is run
  with one test thread.
