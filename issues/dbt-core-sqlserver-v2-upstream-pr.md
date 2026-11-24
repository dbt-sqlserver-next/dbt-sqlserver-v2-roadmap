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

- Local `sqlserver-v2-port` is `51db264c1`, pushed, based on upstream `main`
  `aa8534461`. That upstream merge brought in the `MotherDuck` adapter, the
  `constraint_support` move into `dbt-schemas`, the `RecordBatchExt` move into
  `dbt-adapter-engine` and the new escaping tests.
- The live PR is open/draft at the same head, `51db264c1`. It is mergeable and
  blocked on the required workflow approval, so upstream CI has not run on it
  yet.
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
- **Quoting:** `quote_char = '"'`, as v1 renders and `Fabric` uses; `QUOTED_IDENTIFIER` is ON by default on SQL Server 2022, so no init SQL is needed. Relation components render through the shared `escaped_identifier`, which doubles the T-SQL delimiter, so a name containing `"` renders as `"a""b"` in both `adapter.quote()` and a relation, as in v1.
- **Identifier length:** 128, SQL Server's limit (`sysname` is `nvarchar(128)`), measured; v1 shipped the same 128 in 1.12.3.
- **Auth:** native SQL login is a new `SQLServerAuthIR` variant beside the Entra flows, and an unset `authentication` means `sql`, as in v1. `encrypt`, `TrustServerCertificate` and the connection timeout follow v1's ADBC backend ([dbt-msft/dbt-sqlserver#783](https://github.com/dbt-msft/dbt-sqlserver/pull/783)), checked against a self-signed instance.
- **Schema creation:** `sqlserver__create_schema` and `sqlserver__get_test_sql` pass `target.schema_authorization` into the applock-guarded `create_schema_if_not_exists`, which renders `AUTHORIZATION` only when the profile sets it, as v1's dispatched `sqlserver__create_schema_with_authorization` did. Verified on the live server: the created schema's owner is `guest` with the setting and `dbo` without it.
- **Catalog:** reads `sys.objects`, `sys.schemas`, `sys.columns` and `sys.types`, not `Fabric`'s `sp_tables`/`sp_columns`, which on SQL Server 2022 can't read another database and match `@table_name` as a `LIKE` pattern.
- **Types:** `STRING` → `VARCHAR(MAX)`, v1's native-string default; decimals → `float` or `int` by scale, v1's rule. Column `data_type` keeps lengths as v1's `SQLServerColumn` does (`nvarchar(10)`, `varchar(max)`), and the type formatter never appends `NOT NULL`, since its callers are `CAST` targets.
- **Macro package:** synced with `dbt-sqlserver` 1.12.3 — empty full-refresh tables containing `IDENTITY`, delete+insert target aliasing, `NOT NULL` preservation during widening, quoted snapshot columns, sub-day and week `date_trunc`, plan reuse through `sp_executesql` for the dependent-view lookup and the view skip check, and the backup-relation drop guard. CTE schema probes use `sp_describe_first_result_set` and return a typed empty result; leading `;WITH` is recognized. Three v1 macros with no caller are not vendored: `get_fixture_sql`, `sqlserver__get_unit_test_sql` and `sqlserver__get_relation_without_caching`. `indexes:`, `full_refresh_build: prebuilt` and `table_refresh_method: dml` still raise named compiler errors.
- **Unit tests:** `render_unit_test` has a SQL Server branch. Model CTEs are lifted into the unit-test `WITH` list; `dbt-jinja-utils::inject_and_persist_ephemeral_models` aliases its fallback derived table as required by T-SQL. The expected-schema probe splices leading CTEs into scope and uses an empty typed schema probe for CTE inputs. Empty `given` or `expect` rows emit `WHERE 1 = 0`, since T-SQL has no boolean literal; the predicate branches on `Fabric | SqlServer` so every other dialect keeps `WHERE FALSE`. Upstream commit [`a5d0924`](https://github.com/dbt-labs/dbt/commit/a5d092475163d44c37f70fc4e656f5710f7d8121) warns when effective project/package overrides of `unit` or `get_unit_test_sql` are ignored; Fusion still constructs unit-test SQL in Rust.
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

On local `51db264c1`, upstream parent `aa8534461`:

- `cargo fmt --all --check` and `cargo clippy -p dbt-adapter -p dbt-adapter-core -p dbt-adapter-engine -p dbt-adapter-sql -p dbt-jinja-utils -p dbt-schemas -p dbt-loader -p dbt-tasks-sa -p dbt-sa-cli --all-targets -- -D warnings`: clean
- `cargo check` on the touched adapter, profile, loader, task and CLI crates and `cargo build -p dbt-sa-cli`: clean
- Tests:

  | Crate | Passed |
  |---|---|
  | `dbt-adapter --lib` (LZ4 enabled, one test thread) | 1733 |
  | `dbt-schemas` | 788 |
  | `dbt-auth` | 377 |
  | `dbt-loader` | 66 + 337 |
  | `dbt-tasks-sa --lib` | 334 |
  | `dbt-jinja-utils` | 178 |
  | `dbt-adapter-sql` | 95 |
  | `dbt-adapter-core` | 20 |
  | `dbt-init` | 12 |
  | `dbt-profile-schemas` | 3 |

On SQL Server 2022 `16.0.4295.3` with SQL auth, the rebuilt V2 CLI:

- connects with `dbt debug`;
- builds a full-refresh table from an `IDENTITY` source without giving the target an `IDENTITY` column;
- runs a composite-key delete+insert incremental model twice and preserves nonmatching rows;
- widens a `varchar(5) NOT NULL` column to `varchar(50)` with `on_schema_change: sync_all_columns` and preserves nullability/data;
- adds a reserved-word snapshot column;
- keeps a view's `modify_date` stable across an unchanged run and rebuilds it when its body changes;
- drops a view to build a table under the same name, and back;
- truncates `datetime2` to hour, minute, second and week, and to day/month as before;
- keeps `money` and `smallmoney` columns under an enforced contract;
- types seed columns `bigint`/`varchar(max)` where the data needs it;
- renders a relation whose name contains `"` as `"a""b"`;
- creates a schema owned by `guest` with `schema_authorization: guest`, and `dbo` without it;
- passes a three-model build whose three unit tests include empty `given` and `expect` rows.

Only SQL auth was exercised live; Entra is unit-tested but no Entra-enabled server was available. The adapter suite needs `--features arrow-ipc/lz4` (without it the IPC round-trip test panics) and one test thread; the default parallel run has also exposed shared-state failures in other adapters' tests.

## Known gaps outside this diff

- `dbt.date_spine` fails on SQL Server (nested `WITH`, `order by 1`).
- `dbt_utils.expression_is_true` selects an unaliased `1`, which T-SQL rejects inside the test wrapper.
