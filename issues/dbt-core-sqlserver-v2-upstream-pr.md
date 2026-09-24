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

- `sqlserver-v2-port` (`46704326d`) is 16 commits on `upstream/main`
  `315bad676`: 14 `sqlserver`-scoped commits and the #15766 pair (`f5fedba17`,
  `b8a348ae3`). `fe6b636df` was dropped, since upstream fixed it as `712702b7e`
  (`cell_as_bool`; #15767 closed, #15768 closed unmerged). #15769 reports
  `MERGEABLE`.
- On the rebased branch these all pass: `cargo check --workspace
  --all-targets`, clippy `-D warnings` on the touched crates, and `cargo test`
  (`dbt-adapter --lib` 1517, `dbt-auth` 329, `dbt-schemas` 682, `dbt-loader`
  234, `dbt-tasks-sa` 185, plus `dbt-adapter-core`, `dbt-adapter-sql`,
  `dbt-init` and `dbt-df-providers`). None of it has run against a live server.
- #15766 is still open, and upstream's `execute_inner` still keeps only the last
  batch, so the pair stays until it merges.

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
(Entra + native SQL login), relation quoting, catalog introspection, SQL
type mapping, the `dbt-adapter` match-arm surface, the vendored macro
package, and an optional `dbt init` wizard. Ships gated behind
`DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true` (not added to
`NON_EXPERIMENTAL_ADAPTERS`), same as every other adapter's initial
landing.

Landed as ten sequential, crate-scoped PRs against a staging branch first
(`sqlserver-v2-port` on [dbt-sqlserver-next/dbt-core](https://github.com/dbt-sqlserver-next/dbt-core),
[#11](https://github.com/dbt-sqlserver-next/dbt-core/pull/11)–[#20](https://github.com/dbt-sqlserver-next/dbt-core/pull/20)) —
registering the adapter type makes every exhaustive `match adapter_type()`
across the workspace non-exhaustive, so it can't land upstream piecemeal
without breaking `main`'s build in between. Individual PR descriptions
below cite the specific v1/live-server evidence behind each call; this
summary groups by decision.

## Decisions worth a reviewer's eye

**Quoting**: `quote_char = '"'` with `QUOTED_IDENTIFIER` confirmed ON by
default on a live SQL Server 2022 (no init SQL needed) — matches `Fabric`
and matches what v1 already renders despite T-SQL also accepting
`[brackets]`. `quoted()` doubles an embedded delimiter for `SqlServer`
specifically (the shared default renders `x"q` as unparseable `"x"q"`);
scoped narrowly rather than fixed in the shared default, which changes
rendering for every quoting adapter.

**Identifier length**: `max_identifier_length = 128`, not v1's 127 — SQL
Server's documented and live-verified limit (`sysname` is `nvarchar(128)`);
v1's 127 traces to a copy-pasted Redshift constant, comment included. A
128-character name that v1 rejects builds here; tracked as a v1-side bug
separately, not changed to match v1.

**Auth**: native SQL login (`authentication: sql`) added as a new top-level
`SQLServerAuthIR` variant alongside the existing Entra flows (service
principal, AD password, environment credential — already implemented and
tested pre-port). `encrypt`/`TrustServerCertificate`/`connection timeout`
ported from v1's own ADBC backend
([dbt-msft/dbt-sqlserver#783](https://github.com/dbt-msft/dbt-sqlserver/pull/783),
same driver), live-verified against a self-signed on-prem instance.
Windows/trusted-connection auth and named-instance hosts (`host\instance`)
are deferred — the latter fails loudly at URI parse rather than shipping an
unverified rewrite.

**Catalog introspection**: uses `sys.objects`/`sys.schemas`/`sys.columns`/`sys.types`
directly rather than `Fabric`'s `sp_tables`/`sp_columns` pattern — measured
against live SQL Server 2022 that those procedures are single-database
(cross-database reads, which SQL Server supports and v1 relies on, error
out) and that `@table_name` is a `LIKE` pattern, not an exact match
(`probe_table` vs `probeXtable`). Both are pre-existing gaps in `Fabric`'s
own module too; not touched here since only the pattern half is verifiable
against a T-SQL engine this checkout can reach, and cross-database is
verifiably impossible for Fabric's model to hit at all. Filed as a
follow-up, not fixed in this PR.

**Type mapping**: `STRING` → `VARCHAR(MAX)` (v1's current native-string
default, not `Fabric`'s byte-capped `VARCHAR(8000)` — SQL Server has no
equivalent cap tracked here); decimal → `float`/`int` keyed by scale,
reproducing v1's threshold rather than `Fabric`'s unconditional `float`.

**Macro package**: 34-file v1 tree vendored, mirroring `dbt-fabric`'s
layout. `indexes:` config, `full_refresh_build: prebuilt`, and
`table_refresh_method: dml` all raise a named compiler error rather than
silently no-op'ing or running a different path. Dynamic data masking and
index reconciliation on persisted tables dropped entirely (no v2 Rust
counterpart to call).

**Not registered**: `adapter_specific_behavior_flags` returns `vec![]` —
v1's five behavior flags (native string types, safe type expansion, dbt
transactions, default schema concat, empty relation aliases) all stay on
their v1 default with no alternate Rust code path yet, so nothing is
declared that the adapter can't actually honor.

## Deferred / explicitly out of scope

- Windows/trusted-connection auth
- Named SQL Server instances (`host\instance`)
- Dynamic data masking
- Index/columnstore materialization config (loud error today, not silent)
- `full_refresh_build: prebuilt`, scalar function materializations, table clone support
- `SET XACT_ABORT ON` connection-init SQL (currently off by default; several
  vendored macros assume it's on — v1 issues it, v2 doesn't yet, and there's
  no per-connection init hook to hang it on)
- Collation-aware case folding (`normalize_component` always folds to
  lowercase; wrong under a case-sensitive collation — matches a known,
  skipped-in-CI v1 defect, not a regression)

None of these were silently dropped — each is cited against the specific
v1 behavior or plan decision it diverges from in the individual Part PRs
(`dbt-sqlserver-next/dbt-core` #11–#20) and in
[`05-open-questions-and-risks.md`](https://github.com/dbt-sqlserver-next/dbt-sqlserver-v2-roadmap/blob/main/plan/05-open-questions-and-risks.md)
in the roadmap repo.

## Verified

On this branch, rebased on `main` `315bad676`:

- `cargo check --workspace --all-targets`, `cargo fmt --check`, and
  `cargo clippy --all-targets -D warnings` on the touched crates: clean
- `cargo test -p dbt-adapter --lib`: 1517 passed
- `cargo test -p dbt-auth`: 329 passed; `-p dbt-schemas`: 682; `-p dbt-loader`:
  234; `-p dbt-tasks-sa`: 185; `-p dbt-adapter-sql`: 41; `-p dbt-adapter-core`:
  17; `-p dbt-init`: 7; `-p dbt-df-providers`: 5. No failures.
- `cargo build -p dbt-sa-cli`: clean

Before the rebase, `dbt build` of a T-SQL-ported
[jaffle-shop](https://github.com/dbt-labs/jaffle-shop) (seed, run, tests) ran
against a local SQL Server 2022 container with SQL auth. Unit tests on models
with CTEs fail (see below). Other auth modes are unit-tested in `dbt-auth` only;
no Azure AD credentials were available.

## Known gaps outside this diff

Found while testing, not fixed here; each will be filed on its own:

- Unit tests fail for any model with its own CTEs: the unit-test renderer
  nests the model's `WITH` inside a CTE, which T-SQL rejects (Msg 156).
- `dbt.date_spine` fails on SQL Server (nested `WITH`, `order by 1`); the fix
  is a `sqlserver__date_spine` override in the adapter's macros.
- `dbt_utils.expression_is_true` selects an unaliased `1`, which T-SQL rejects
  inside the test wrapper (dbt-labs/dbt-utils).

## Note on two out-of-scope commits in this diff

This branch also carries `f5fedba17`/`b8a348ae3`, a multi-statement query batch
fix found while smoke-testing this adapter. It's a general engine bug, filed and
reviewable on its own as
[#15765](https://github.com/dbt-labs/dbt/issues/15765)/[#15766](https://github.com/dbt-labs/dbt/pull/15766).
Please review it there; it drops out of this diff once #15766 merges.
