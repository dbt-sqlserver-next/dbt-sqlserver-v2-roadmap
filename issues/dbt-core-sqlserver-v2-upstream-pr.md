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

- Local `sqlserver-v2-port` is `bad853106`, merging port head `4d70a106` with
  `dbt-labs/dbt:main` `b601ffe4`. The merge includes upstream's 2.0.8-era API
  refactors, including boxed `Column` constructors and generic profile setup.
- The live PR is still open/draft at remote head `4d70a106` and reports a dirty
  merge state. The local refresh has not been pushed; upstream CI has not run
  on it.
- The branch still includes fork PRs [#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21)
  through [#26](https://github.com/dbt-sqlserver-next/dbt-core/pull/26). The
  #15766 pair remains dropped; #15766 and #15765 are closed.

## Notes for filing

**Changelogs.** `.changes/unreleased/Features-20260803-sqlserver-adapter.yaml`
records the adapter; `.changes/unreleased/Fixes-20261031-045025.yaml` records
the current correctness and CTE schema fixes.

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
- **Macro package:** based on v1's 1.12.0 sync, with follow-up fixes for empty full-refresh tables containing `IDENTITY`, delete+insert target aliasing, `NOT NULL` preservation during widening, and quoted snapshot columns. CTE schema probes use `sp_describe_first_result_set` and return a typed empty result; leading `;WITH` is recognized. `indexes:`, `full_refresh_build: prebuilt` and `table_refresh_method: dml` still raise named compiler errors.
- **Unit tests:** `render_unit_test` has a SQL Server branch. Model CTEs are lifted into the unit-test `WITH` list; `dbt-jinja-utils::inject_and_persist_ephemeral_models` now aliases its fallback derived table as required by T-SQL. The expected-schema probe splices leading CTEs into scope and uses an empty typed schema probe for CTE inputs.
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

On local merge commit `bad853106`, first parent `4d70a106` and upstream parent `b601ffe4`:

- `cargo fmt --check` on touched crates and `cargo clippy -p dbt-adapter -p dbt-adapter-engine -p dbt-init -p dbt-profile-schemas -p dbt-jinja-utils -p dbt-loader -p dbt-tasks-sa -p dbt-sa-cli --all-targets -- -D warnings`: clean
- `cargo check` on the touched adapter, profile, loader, task and CLI crates and `cargo build -p dbt-sa-cli`: clean
- Tests:

  | Crate | Passed |
  |---|---|
  | `dbt-adapter --lib` (LZ4 enabled, one test thread) | 1588 |
  | `dbt-schemas` | 708 |
  | `dbt-auth` | 353 |
  | `dbt-loader` | 258 |
  | `dbt-tasks-sa` | 285 |
  | `dbt-jinja-utils` | 161 |
  | `dbt-adapter-sql` | 46 |
  | `dbt-adapter-core` | 20 |
  | `dbt-init` | 7 |
  | `dbt-profile-schemas` | 3 |

On SQL Server 2022 `16.0.4295.3` with SQL auth, the rebuilt V2 CLI:

- connects with `dbt debug`;
- builds a full-refresh table from an `IDENTITY` source without giving the target an `IDENTITY` column;
- runs a composite-key delete+insert incremental model twice and preserves nonmatching rows;
- widens a `NOT NULL` column with `on_schema_change: sync_all_columns` and preserves nullability/data;
- runs the four-step type rewrite and removes a leftover temp column while preserving nullability/data;
- adds a reserved-word snapshot column from a semicolon-prefixed CTE;
- runs a non-leading-`WITH` ephemeral reference and a CTE schema-only probe without evaluating a divide-by-zero expression.

Only SQL auth was exercised live; Entra is unit-tested but no Entra-enabled server was available. The default parallel adapter test run also exposed an unrelated Databricks catalogs-v2 shared-state failure; the serial run with Arrow IPC LZ4 enabled passed all adapter tests.

## Known gaps outside this diff

- `dbt.date_spine` fails on SQL Server (nested `WITH`, `order by 1`).
- `dbt_utils.expression_is_true` selects an unaliased `1`, which T-SQL rejects inside the test wrapper.
