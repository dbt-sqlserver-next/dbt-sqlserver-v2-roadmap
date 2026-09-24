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

Verified against the checkout: `sqlserver-v2-port` is 16 commits ahead of
`upstream/main` (`git log --oneline upstream/main..HEAD`), 14 of them
`sqlserver`-scoped (the Part 1–10 series plus the changelog entry added
below), plus two general-engine fixes found while smoke-testing Part 10.

## Notes for filing

**1. Two commits overlap an already-open, separate upstream PR — stated in
the PR body below, not resolved by a rebase.** `16152a55f` (`fix(adapter):
keep the real result set from a multi-statement query batch`) and
`2343c50b4` (its changelog entry) are already filed independently:
[dbt-labs/dbt#15765](https://github.com/dbt-labs/dbt/issues/15765)
/ [PR #15766](https://github.com/dbt-labs/dbt/pull/15766). #15766 is
still open and unmerged, now with merge conflicts against `main` and no
maintainer review, so there's nothing to rebase away yet. The bug is still
present on `upstream/main` (`execute_inner` in `adapter_impl.rs` keeps the
last statement's batch). They stay in this diff; the PR body says so explicitly so reviewers
treat them as already-tracked at `#15765`/`#15766` rather than re-reviewing
them here. Once `#15766` merges, the next `sqlserver-v2-port` sync with
`main` drops both commits automatically (no action needed then either).

**2. The `should_warn`/`should_error` truthiness fix — fixed upstream; drop
`fe6b636df`.** The maintainers fixed it independently in `712702b7e`
(`fix(tasks): coerce text 'true'/'false' in test-result boolean columns`),
closed #15767 as completed, and #15768 was closed unmerged. `fe6b636df` on
`sqlserver-v2-port` is now redundant, conflicts with `712702b7e` in
`materialize.rs`, and has to come out of this branch at the next sync with
`main`. How it was filed: no existing
upstream issue (checked `gh issue list --repo dbt-labs/dbt --search
"should_warn should_error truthiness"` and related terms, nothing hit).
Filed from the drafted content
(`issues/v2-test-result-bool-parsing-truthiness.md`), trimmed to match the
sibling issue's (`#15765`) Summary/Reproduction/Suggested-fix shape:
[dbt-labs/dbt#15767](https://github.com/dbt-labs/dbt/issues/15767),
and the fix cherry-picked to a `main`-based branch
(`upstream-fix-should-warn-truthiness`) and opened as
[PR #15768](https://github.com/dbt-labs/dbt/pull/15768) using the
repo's actual `pull_request_template.md` (`Resolves #` / `Problem` /
`Solution` / `Checklist`), with its own changelog entry
(`Fixes-20260803-should-warn-truthiness.yaml`). `fe6b636df` on
`sqlserver-v2-port` was kept in the branch with the same "found while
smoke-testing, tracked separately" treatment as item 1.

**3. `Features` changelog entry — created.**
`.changes/unreleased/Features-20260803-sqlserver-adapter.yaml` added and
committed (`5085525a8`):

```yaml
kind: Features
body: Add experimental support for the SQL Server adapter (dbt Fusion / v2). Enable
    with DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true.
time: 2026-08-03T00:00:00.000000-00:00
custom:
    author: axellpadilla
    issue: "15714"
    project: dbt-core
```

**4. The branch is behind `main` and doesn't merge cleanly.** #15769 shows
as conflicting. `upstream/main` is 720 commits past the branch's merge-base
(`a5fe6f429`), and a trial `git merge-tree` conflicts in 14 files, among them
`adapter_impl.rs`, `relation_impl.rs`, `sql_types.rs`,
`dbt-auth/src/sqlserver/mod.rs` and `dbt-profile-schemas/src/lib.rs`. Upstream
also moved `dbt-init/src/adapter_config/` into `dbt-profile-schemas/`, where
`sqlserver_config.rs` has to follow, and inlined
`format_ident::default_identifier_case` into `normalize_component`
(`f98cbd643`), so the branch's `SqlServer` comment on that call has to move
onto the inline match.

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

- `cargo build -p dbt-adapter-core -p dbt-adapter-sql`: clean
- `cargo nextest run -p dbt-schemas`: 446 passed / 0 failed
- `cargo nextest run -p dbt-auth`: 287 passed / 0 failed
- `cargo test -p dbt-adapter --lib`: 891 passed / 0 failed
- `cargo test -p dbt-loader --test main`: 47 passed / 0 failed (43 pre-existing + 4 new)
- `cargo test -p dbt-init`: 7/7 passing (no regression; no new tests — no
  `adapter_config/*.rs` file in the crate has coverage today, `fabric_config.rs` included)
- `cargo fmt --check` / `cargo clippy --all-targets`: clean throughout
- `cargo build -p dbt-sa-cli`: clean
- End-to-end: clean `dbt build` (seed, run, generic tests, unit tests)
  against a local SQL Server 2022 container and a T-SQL-ported
  [jaffle-shop](https://github.com/dbt-labs/jaffle-shop), plain SQL auth.
  Auth-mode matrix beyond plain SQL (service principal, AD password,
  environment credential) not exercised end-to-end — no Azure AD
  credentials available in this environment; each is unit-tested in
  `dbt-auth` individually.

## Bugs found outside SQL Server's own code, filed separately

Two pre-existing gaps shared with `Fabric` (same T-SQL engine, same
unmodified shared macros), not fixed here: `dbt_utils.expression_is_true`
selects an unaliased literal, which T-SQL rejects from any named derived
table or view; `dbt.date_spine`/`generate_series` nests a `WITH` block
inside another CTE, which T-SQL also rejects, breaking
`metricflow_time_spine`. Roadmap-repo drafts:
`issues/v2-dbt-utils-expression-is-true-unnamed-column-tsql.md`,
`issues/v2-date-spine-nested-cte-tsql.md`.

## Note on two out-of-scope commits in this diff

This branch also carries `16152a55f`/`2343c50b4` (a multi-statement query
batch fix) and `fe6b636df` (a test-boolean-parsing fix) — general dbt
Fusion engine bugs found while smoke-testing this adapter, not part of the
SQL Server port itself. Each is filed and reviewable on its own:
[#15765](https://github.com/dbt-labs/dbt/issues/15765)/[#15766](https://github.com/dbt-labs/dbt/pull/15766)
for the first, [#15767](https://github.com/dbt-labs/dbt/issues/15767)/[#15768](https://github.com/dbt-labs/dbt/pull/15768)
for the second. Please review those separately rather than as part of this
adapter's diff. The second is already fixed on `main` by `712702b7e` and
leaves this branch at its next sync; the first drops out once `#15766`
merges.
