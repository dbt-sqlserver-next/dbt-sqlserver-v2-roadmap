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

- `sqlserver-v2-port` (`fa6731a42`) is on `upstream/main` `315bad676`, with
  fork PRs [#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21) to
  [#25](https://github.com/dbt-sqlserver-next/dbt-core/pull/25) merged. #25
  synced the macros with dbt-sqlserver 1.12.0 and widens columns with a single
  `ALTER COLUMN` through `sqlserver__expand_target_column_types`. The #15766
  pair was dropped; #15766 and #15765 are closed.
- It merges into `upstream/main` `242065240` without conflicts. On `main`
  `3d61704d4` (with #25's first commit) the merged tree also passed clippy,
  the SQL Server crates' tests and a live build.
- Fork PR [#26](https://github.com/dbt-sqlserver-next/dbt-core/pull/26)
  (open) lifts a model's own CTEs into its unit test's `WITH` list, fixing
  unit tests on models with their own `WITH` (and the schema probe for
  ephemeral givens) without v1's views.
  Once it merges, the body's "Known gaps" loses that line and "Decisions" gains
  the `render_unit_test` branch.
- The live body was synced on 2026-09-28 and matches the draft below, with
  "Verified" re-run on `fa6731a42`.
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

Adds `AdapterType::SqlServer` to dbt Fusion: profile config, authentication
(Entra and native SQL login), relation quoting, catalog introspection, SQL
type mapping, the `dbt-adapter` match-arm surface, the vendored macro
package, and an optional `dbt init` wizard. It ships gated behind
`DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true` (not added to
`NON_EXPERIMENTAL_ADAPTERS`), like every other adapter's first landing.

The work landed on a staging branch first (`sqlserver-v2-port` on
[dbt-sqlserver-next/dbt-core](https://github.com/dbt-sqlserver-next/dbt-core)):
ten crate-scoped PRs
([#11](https://github.com/dbt-sqlserver-next/dbt-core/pull/11)–[#20](https://github.com/dbt-sqlserver-next/dbt-core/pull/20)),
then fixes found by running it against SQL Server and comparing it with
dbt-sqlserver 1.12.0
([#21](https://github.com/dbt-sqlserver-next/dbt-core/pull/21)–[#25](https://github.com/dbt-sqlserver-next/dbt-core/pull/25)).
Registering the adapter type makes every exhaustive `match adapter_type()`
non-exhaustive, so it can't land upstream piecemeal without breaking
`main`'s build in between. Each fork PR cites the v1 and live-server
evidence behind its calls; this summary groups them by decision.

## Decisions worth a reviewer's eye

**Batches**: `execute_inner` sends a SQL Server `statement()` block whole,
as BigQuery, DuckDB and LakeCompute already are, instead of splitting it on
`;`. T-SQL scopes a `DECLARE`d variable to its batch and `MERGE` needs its
terminator, so splitting broke every MERGE incremental rerun (Msg 10713) and
`persist_docs`/`drop_schema` (Msg 137). v1 sets `XACT_ABORT ON` on every
connection, and v2 has no per-connection hook, so `SET XACT_ABORT ON` runs as
its own statement on the same connection just before each batch. A failed
statement then stops the rest of its batch, as in v1. It isn't prefixed to
the batch, because a `CREATE VIEW` must open its batch (Msg 111).

**Quoting**: `quote_char = '"'`. `QUOTED_IDENTIFIER` is ON by default on
SQL Server 2022, so no init SQL is needed. This matches `Fabric` and what v1
renders, although T-SQL also accepts `[brackets]`. `quoted()` doubles an
embedded delimiter for `SqlServer` only (the shared default renders `x"q` as
unparseable `"x"q"`). Fixing the shared default would change rendering for
every quoting adapter.

**Identifier length**: `max_identifier_length = 128`, not v1's 127. That's
SQL Server's documented and measured limit (`sysname` is `nvarchar(128)`);
v1's 127 is a Redshift constant copied with its comment. A 128-character
name that v1 rejects builds here.

**Auth**: native SQL login (`authentication: sql`) is a new top-level
`SQLServerAuthIR` variant beside the existing Entra flows (service
principal, AD password, environment credential, access token). An unset
`authentication` means `sql`, as in v1. `encrypt`,
`TrustServerCertificate` and the connection timeout are ported from v1's
ADBC backend
([dbt-msft/dbt-sqlserver#783](https://github.com/dbt-msft/dbt-sqlserver/pull/783),
same driver), and were checked against a self-signed on-prem instance.

**Catalog introspection**: reads `sys.objects`/`sys.schemas`/`sys.columns`/`sys.types`
directly rather than `Fabric`'s `sp_tables`/`sp_columns`. Measured on SQL
Server 2022: those procedures can't read another database, which SQL Server
supports and v1 relies on, and `@table_name` is a `LIKE` pattern
(`probe_table` also matches `probeXtable`). `Fabric`'s module has the same
pattern gap and isn't touched here.

**Types**: `STRING` → `VARCHAR(MAX)` (v1's native-string default, not
`Fabric`'s `VARCHAR(8000)`). Decimal → `float`/`int` keyed by scale, v1's
threshold. Column `data_type` follows v1's `SQLServerColumn`: `nvarchar(10)`,
`varchar(max)` and `char(3)` keep their length, where the generic handling
gave a bare `nvarchar` (length 1 in DDL). The type formatter never appends
`NOT NULL`, since its callers are `CAST` targets and `Column.dtype`.

**Macro package**: v1's tree is vendored in `dbt-fabric`'s layout and synced
with dbt-sqlserver 1.12.0. Changes from v1:

- Calls the Rust adapter lacks (`commit_if_open`, `max_rows`) are gone;
  v2 has no ambient transaction.
- Columns widen through `sqlserver__expand_target_column_types`, which
  passes `prefer_single` to `alter_column_type`, as v1's Python does. A
  widening takes a single `ALTER COLUMN`, which keeps the column's indexes,
  default and position. `on_schema_change` keeps the four-step rewrite, as
  in v1.
- `indexes:`, `full_refresh_build: prebuilt` and
  `table_refresh_method: dml` raise a named compiler error instead of
  running a different path. Dynamic data masking and denies aren't ported;
  they need `adapter.resolve_*`.

**Not registered**: `adapter_specific_behavior_flags` returns `vec![]`.
v1's behavior flags have no alternate v2 code path, so each stays at its
v1 default, and nothing is declared that the adapter can't honor.

## Deferred / explicitly out of scope

- Windows/trusted-connection auth, and the profile keys `windows_login`,
  `xact_abort` (always on) and `backend`
- Named SQL Server instances (`host\instance`): fails at URI parse
- Dynamic data masking
- Custom `indexes:` config (a compiler error, not a silent no-op)
- `full_refresh_build: prebuilt`, scalar function materializations, table clone
- Collation-aware case folding: `normalize_component` always folds to
  lowercase, which is wrong under a case-sensitive collation. v1 has the same
  defect.
- Setting `prefer_single_alter_column`: the key isn't in the shared config
  schema, so it's rejected (dbt1060). The macros read it once it's accepted.

Each is cited against the v1 behavior or plan decision it diverges from in
the fork PRs and in
[`05-open-questions-and-risks.md`](https://github.com/dbt-sqlserver-next/dbt-sqlserver-v2-roadmap/blob/main/plan/05-open-questions-and-risks.md).

## Verified

On `fa6731a42` (based on `main` `315bad676`; it merges into `main`
`242065240` without conflicts):

- `cargo fmt --check`, and `cargo clippy --all-targets -D warnings` on the
  ten touched crates: clean
- Tests, all passing:

  | Crate | Passed |
  |---|---|
  | `dbt-adapter --lib` | 1518 |
  | `dbt-schemas` | 683 |
  | `dbt-auth` | 329 |
  | `dbt-loader` | 243 |
  | `dbt-tasks-sa` | 185 |
  | `dbt-adapter-sql` | 41 |
  | `dbt-adapter-core` | 17 |
  | `dbt-init` | 7 |
  | `dbt-df-providers` | 5 |
  | `dbt-profile-schemas` | 1 |

- `cargo build -p dbt-sa-cli`: clean

On SQL Server 2022 (16.0.4295.3) with SQL auth, a scratch project builds with
`--full-refresh` and again without, 8/8 each time: a `check` snapshot over a
`WITH` query, two incremental models, views, a unit test, and generic tests.
On earlier commits of this branch (the fork PRs list each run):

- **Generic tests:** passing, warning and failing ones are reported correctly,
  and so are a singular test and `store_failures`.
- **Incremental models:** a MERGE rerun, a widened column on rerun (on the
  tip too, including one with an index and a default), and a type change
  under `on_schema_change: sync_all_columns`.
- **Other:** `persist_docs`, `drop_schema`, `run-operation` writes, and views
  that are unchanged or have a changed source.

Other auth modes are unit-tested in `dbt-auth` only; no Entra-enabled
server was available.

## Known gaps outside this diff

Found while testing, not fixed here:

- Unit tests fail for any model with its own CTEs: the unit-test renderer
  nests the model's `WITH` inside a CTE, which T-SQL rejects (Msg 156).
- A model that refs an ephemeral model and doesn't start with `WITH` fails:
  the shared CTE injection wraps it in `select * from ( … )` with no alias
  (Msg 102).
- `dbt.date_spine` fails on SQL Server (nested `WITH`, `order by 1`); the fix
  is a `sqlserver__date_spine` override in the adapter's macros.
- `dbt_utils.expression_is_true` selects an unaliased `1`, which T-SQL rejects
  inside the test wrapper (dbt-labs/dbt-utils).
