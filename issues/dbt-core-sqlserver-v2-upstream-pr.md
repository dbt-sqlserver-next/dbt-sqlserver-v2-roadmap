---
target_repo: dbt-labs/dbt
type: pr-draft
status: open
url: https://github.com/dbt-labs/dbt/pull/15769
base: main
head: dbt-sqlserver-next/dbt-core:sqlserver-v2-port
closes: https://github.com/dbt-labs/dbt/issues/15714
related: ../plan/README.md
---

# Draft: PR body for `sqlserver-v2-port` → `dbt-labs/dbt:main`

Per `plan/README.md` "Branching strategy": Part 10 closing is what triggers
this. [Part 10 is closed](https://github.com/dbt-sqlserver-next/dbt-core/issues/10)
(merged as [PR #20](https://github.com/dbt-sqlserver-next/dbt-core/pull/20)),
so the branch is at the point where this was supposed to get opened.

## Branch state

- `sqlserver-v2-port` (`4d70a1069`) is on `upstream/main` `315bad676`, with
  fork PRs [#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21) to
  [#26](https://github.com/dbt-sqlserver-next/dbt-core/pull/26) merged. #26
  moves a model's own CTEs into its unit test's `WITH` list. The #15766 pair
  was dropped; #15766 and #15765 are closed.
- It merges into `upstream/main` `242065240` without conflicts.
- The live body matches the draft below.
- Upstream CI hasn't run on #15769: it's a public fork and needs the
  `ci:approve-public-fork-ci` label from a maintainer. CodeScene's one critical
  finding is `SqlServerDbConfig::set_field`, the same per-field `match` as
  Fabric's.

## Notes for filing

**Changelog.** `.changes/unreleased/Features-20260803-sqlserver-adapter.yaml`
(`kind: Features`, `issue: "15714"`) is committed on the branch.

## Draft PR

**Title:** `feat(sqlserver): add dbt Fusion (v2) adapter support`

**Base:** `dbt-labs/dbt:main` · **Head:** `dbt-sqlserver-next/dbt-core:sqlserver-v2-port`

---

Closes #15714.

## Summary

Adds `AdapterType::SqlServer` to dbt Fusion: profile config, authentication (Entra and native SQL login), quoting, catalog introspection, type mapping, the `dbt-adapter` match arms, the vendored macro package and a `dbt init` wizard. It ships behind `DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true`, like every adapter's first landing.

```
fork PRs #11–#20 (one crate each) ───────────────────────┐
fork PRs #21–#26 (fixes vs SQL Server 2022 and v1 1.12.0) ┤
                                                          ▼
dbt-sqlserver-next/dbt-core:sqlserver-v2-port ──► this PR ──► dbt-labs/dbt:main
```

Registering the adapter type makes every exhaustive `match adapter_type()` non-exhaustive, so it can't land upstream piecemeal. The fork PRs ([#11](https://github.com/dbt-sqlserver-next/dbt-core/pull/11)–[#26](https://github.com/dbt-sqlserver-next/dbt-core/pull/26)) hold the evidence behind each call below.

## Decisions worth a reviewer's eye

- **Batches:** `execute_inner` sends a SQL Server `statement()` block whole, like BigQuery, DuckDB and LakeCompute, after `SET XACT_ABORT ON` on the same connection. Splitting on `;` broke MERGE reruns (Msg 10713) and `persist_docs`/`drop_schema` (Msg 137, since `DECLARE` is batch-scoped). `XACT_ABORT` stops a batch at its first error, as v1's per-connection setting does. It runs as its own statement because a `CREATE VIEW` must open its batch (Msg 111).
- **Quoting:** `quote_char = '"'`, as v1 renders and `Fabric` uses; `QUOTED_IDENTIFIER` is ON by default on SQL Server 2022, so no init SQL is needed. `quoted()` doubles an embedded `"` for `SqlServer` only, since changing the shared default would change every quoting adapter.
- **Identifier length:** 128, SQL Server's limit (`sysname` is `nvarchar(128)`), measured; v1's 127 is a copied Redshift constant.
- **Auth:** native SQL login is a new `SQLServerAuthIR` variant beside the Entra flows, and an unset `authentication` means `sql`, as in v1. `encrypt`, `TrustServerCertificate` and the connection timeout follow v1's ADBC backend ([dbt-msft/dbt-sqlserver#783](https://github.com/dbt-msft/dbt-sqlserver/pull/783)), checked against a self-signed instance.
- **Catalog:** reads `sys.objects`, `sys.schemas`, `sys.columns` and `sys.types`, not `Fabric`'s `sp_tables`/`sp_columns`, which on SQL Server 2022 can't read another database and match `@table_name` as a `LIKE` pattern.
- **Types:** `STRING` → `VARCHAR(MAX)`, v1's native-string default; decimals → `float` or `int` by scale, v1's rule. Column `data_type` keeps lengths as v1's `SQLServerColumn` does (`nvarchar(10)`, `varchar(max)`), and the type formatter never appends `NOT NULL`, since its callers are `CAST` targets.
- **Macro package:** v1's, synced with dbt-sqlserver 1.12.0, without the calls the Rust adapter lacks (`commit_if_open`, `max_rows`). Widening goes through `sqlserver__expand_target_column_types`, which passes `prefer_single` as v1's Python does, so a single `ALTER COLUMN` keeps the column's indexes and default. `indexes:`, `full_refresh_build: prebuilt` and `table_refresh_method: dml` raise a named compiler error.
- **Unit tests:** `render_unit_test` gets a `SqlServer` branch beside its DuckDB, ClickHouse, Snowflake, BigQuery and Databricks ones. T-SQL rejects the model's own `WITH` inside `<model>_actual` (Msg 156), and v2 doesn't run v1's `sqlserver__get_unit_test_sql` (#16504), so the model's CTEs move into the unit test's `WITH` list, split off with `split_leading_ctes` (`dbt-jinja-utils`). The expected-schema probe splices them the same way, because `sqlserver__get_empty_subquery_sql` drops a `WITH` header.
- **Behavior flags:** none registered. v1's flags have no alternate v2 code path, so each stays at its v1 default.

## Deferred / explicitly out of scope

- Windows/trusted-connection auth, and the profile keys `windows_login`, `xact_abort` (always on) and `backend`
- Named instances (`host\instance`), which fail at URI parse
- Dynamic data masking and denies, which need `adapter.resolve_*`
- Custom `indexes:`, `full_refresh_build: prebuilt`, scalar function materializations and table clone
- Collation-aware case folding: `normalize_component` always lowercases, which is wrong under a case-sensitive collation, as in v1
- Setting `prefer_single_alter_column`, which the shared config schema rejects (dbt1060); the macros read it once it's accepted

Each is cited against the v1 behavior it diverges from in the fork PRs and in [`05-open-questions-and-risks.md`](https://github.com/dbt-sqlserver-next/dbt-sqlserver-v2-roadmap/blob/main/plan/05-open-questions-and-risks.md).

## Verified

On `4d70a1069`, based on `main` `315bad676`; it merges into `main` `242065240` without conflicts. Crates that #26 didn't touch were last run on `fa6731a42`, which differs only in `dbt-tasks-sa` and `dbt-jinja-utils`.

- `cargo fmt --check`, and `cargo clippy --all-targets -D warnings` on the touched crates: clean
- `cargo build -p dbt-sa-cli`: clean
- Tests, all passing:

  | Crate | Passed |
  |---|---|
  | `dbt-adapter --lib` | 1518 |
  | `dbt-schemas` | 683 |
  | `dbt-auth` | 329 |
  | `dbt-loader` | 243 |
  | `dbt-tasks-sa` | 188 |
  | `dbt-jinja-utils` | 130 |
  | `dbt-adapter-sql` | 41 |
  | `dbt-adapter-core` | 17 |
  | `dbt-init` | 7 |
  | `dbt-df-providers` | 5 |
  | `dbt-profile-schemas` | 1 |

On SQL Server 2022 (16.0.4295.3) with SQL auth:

- **A scratch project** builds 8/8 with `--full-refresh` and again without: a `check` snapshot over a `WITH` query, two incremental models, views, a unit test and generic tests.
- **Unit tests** on models with their own CTEs pass, including one with an ephemeral input, and a wrong expectation fails with the right diff.
- **Earlier commits of this branch,** as listed in each fork PR:
  - generic tests that pass, warn and fail, a singular test and `store_failures`;
  - a MERGE rerun, a widened column on an indexed and defaulted column, and `on_schema_change: sync_all_columns`;
  - `persist_docs`, `drop_schema`, `run-operation` writes, and unchanged and stale views.

Other auth modes are unit-tested in `dbt-auth` only; no Entra-enabled server was available.

## Known gaps outside this diff

- A model that refs an ephemeral model and doesn't start with `WITH` fails: the shared CTE injection wraps it in an unaliased derived table (Msg 102), #16505.
- `dbt.date_spine` fails on SQL Server (nested `WITH`, `order by 1`).
- `dbt_utils.expression_is_true` selects an unaliased `1`, which T-SQL rejects inside the test wrapper.
