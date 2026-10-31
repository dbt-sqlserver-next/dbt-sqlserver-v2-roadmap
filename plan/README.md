# dbt-sqlserver on dbt Core v2 (Fusion / Rust engine) — Implementation Plan

Root plan for porting the community **dbt-sqlserver** adapter (currently a
standalone v1 Python package at [`dbt-sqlserver/`](../dbt-sqlserver)) into
**dbt Core v2.0**, the Rust monorepo checked out at [`dbt-core/`](../dbt-core).

This plan follows the official contribution guide:

- **Guide**: https://docs.getdbt.com/guides/adapter-creation-v2
- **Upgrading to v2**: https://docs.getdbt.com/docs/dbt-versions/core-upgrade/upgrading-to-v2?version=2.0 —
  what v2 changes for everyone, not just adapter authors: the supported-adapter
  list SQL Server is absent from, parse-time strictness, Jinja and `config`
  semantics that break v1 macros, per-adapter threading, `--use-v2-parser`.
  The macro-facing parts are summarized in `03-macros-porting-map.md`.
- **Fusion package compatibility**: https://docs.getdbt.com/guides/fusion-package-compat —
  the maintainer-side flow for dbt packages (`dbt-autofix`, `dbtf build`,
  `require-dbt-version: [">=1.10.0,<3.0.0"]`). Relevant because dbt-sqlserver
  users need it from every package in their project, not because it applies to
  this port.

  **Packages and adapters are distributed differently in v2, and the word
  "package" invites the wrong assumption.** Hub packages ship independently, as
  they always have — no dbt-core PR. Adapters don't: the adapter-creation guide
  is explicit that there's "no `setup.py`, no PyPI release — dbt Labs ships your
  adapter once the PR is merged." That follows from the code, where adapter
  behavior is exhaustive `match adapter_type()` arms across nine crates plus a
  RustEmbed'd macro package, all compiled into the binary. The one dynamic
  loading boundary in dbt-core is `crates/dbt-adbc`, which loads driver
  `.so`/`.dylib` files — so an adapter's *connectivity* is pluggable (SQL
  Server's driver already ships from the CDN, `00-current-state.md` §2) while
  its *semantics* are not. In v1 the adapter was itself a pip package, which is
  where the confusion comes from.
- **Target repo (upstream)**: https://github.com/dbt-labs/dbt (local checkout: `dbt-core/`)
- **Working fork + integration branch**: https://github.com/dbt-sqlserver-next/dbt-core,
  branch `sqlserver-v2-port`. All contributor work lands here first, not
  directly on upstream `main` — see "Branching strategy" below for why.
- **v1 source to port from**: https://github.com/dbt-msft/dbt-sqlserver (local checkout: `dbt-sqlserver/`)
- **ADBC driver (already exists, CDN-distributed)**: `adbc_driver_mssql`, built on
  [microsoft/go-mssqldb](https://github.com/microsoft/go-mssqldb)
- **Same driver, working reference implementation**: v1's `adbc` backend
  ([PR #783](https://github.com/dbt-msft/dbt-sqlserver/pull/783), commit
  `bca9e3e`) connects to `adbc_driver_mssql` via the same `sqlserver://` URI
  scheme v2 uses. Its `sqlserver_backend.py`/`sqlserver_credentials.py` are a
  merged, working reference for the connection parameters (`encrypt`,
  `TrustServerCertificate`, `connection timeout`, named-instance handling) that
  v2's `crates/dbt-auth/src/sqlserver/mod.rs` still carries as `// TODO`s —
  `00-current-state.md` §3.
- **Reference adapter for T-SQL family patterns**: Microsoft Fabric
  (`AdapterType::Fabric`), already implemented in this checkout — Fabric Warehouse
  and SQL Server share the T-SQL dialect and the same `go-mssqldb`/ADBC wire driver,
  so most of the Rust match arms can be written as a sibling of the `Fabric` arm
  rather than from scratch.
- **v1 Python adapter reference**: [dbt-fabric](https://github.com/microsoft/dbt-fabric)
  (already cited in-repo as the source for auth semantics) and
  [dbt-msft/dbt-sqlserver](https://github.com/dbt-msft/dbt-sqlserver) (this repo's
  `dbt-sqlserver/` checkout) for macro/catalog SQL to port.

## Current implementation status

The port is implemented on the local `dbt-core:sqlserver-v2-port` branch. See
[`00-current-state.md`](00-current-state.md) for the current commit, verification,
and remaining gaps.

| Piece | Status | Evidence |
|---|---|---|
| Adapter registration and profile | ✅ Implemented | `AdapterType::SqlServer`, `SqlServerDbConfig`, profile setup and init wizard |
| Connectivity | ✅ Implemented | SQL Server ADBC backend using MSSQL driver `1.7.1`; SQL login and supported Entra flows |
| Adapter runtime | ✅ Implemented | Relation/catalog introspection, type mapping, column builder, adapter method arms |
| Macro package | ✅ Implemented | Vendored SQL Server macros with current RC refresh fixes |
| Experimental gate | ✅ Implemented | `DBT_ALLOW_EXPERIMENTAL_ADAPTERS=true` |

The remaining work is PR refresh/review, CI on the updated upstream base, and
closing or explicitly documenting the compatibility gaps in `05`.

## Plan structure

This plan is split into files because the full scope doesn't fit comfortably in
one document:

1. [`00-current-state.md`](00-current-state.md) — current audit of the local port, verified upstream base, tests, and remaining gaps.
2. [`01-architecture-and-prerequisites.md`](01-architecture-and-prerequisites.md) — v2 architecture recap, SQL Server-specific warehouse knowledge needed, dev machine setup.
3. [`02-implementation-steps.md`](02-implementation-steps.md) — current implementation status, validation and remaining release gaps.
4. [`03-macros-porting-map.md`](03-macros-porting-map.md) — table mapping every v1 Jinja macro in `dbt-sqlserver/dbt/include/sqlserver/macros/` to its v2 destination and porting notes.
5. [`04-testing-and-validation.md`](04-testing-and-validation.md) — type-checking, smoke testing, jaffle-shop acceptance bar, CI coordination process, SQL Server-specific test matrix (auth modes, on-prem vs Azure SQL DB vs Azure SQL MI vs Fabric-adjacent).
6. [`05-open-questions-and-risks.md`](05-open-questions-and-risks.md) — the settled decisions (quoting, auth surface, TLS, deferred scope, smoke-test target) and the known risk areas.

## Branching strategy

Registering `AdapterType::SqlServer` breaks exhaustive `match adapter_type()`
blocks workspace-wide, so keep the implementation on the integration branch
until the full slice builds. The local integration now includes upstream
`main` `b601ffe4`; the live upstream PR still points at the previous fork head
until the branch is pushed and PR CI is approved.

- **Fork**: https://github.com/dbt-sqlserver-next/dbt-core
- **Integration branch**: `sqlserver-v2-port`
- PRs (including from dbt-sqlserver contributors) target
  `dbt-sqlserver-next/dbt-core:sqlserver-v2-port`.
- Upstream contribution remains one atomic adapter PR (or a tight same-day
  sequence) to `dbt-labs/dbt:main`.
- Merge upstream `main` before each review cycle; verify the exact base SHA and
  run adapter tests and the SQL Server live matrix before updating the PR.

## Why `dbt-core/` and `dbt-sqlserver/` aren't submodules

Both are `.gitignore`d here and cloned via the `Makefile` (`make setup`) rather
than tracked as submodules. This was revisited when the repo went public and
became a shared coordination point: submodules would give collaborators a
pinned, reproducible checkout, which is normally what a public collaboration
repo wants. Two things outweigh it:

- **The audience for `plan/` mostly doesn't need the code checked out at
  all.** Submodules add mandatory `git submodule update --init` friction to
  a repo whose primary use is "read the roadmap."
- **Both repos are actively being patched, not consumed as frozen
  dependencies.** Commits get pushed to forks of `dbt-core`/`dbt-sqlserver`
  as implementation work happens; a submodule pointer would need bumping on
  every such push, on both repos — exactly the overhead a plain `git clone`
  avoids. The `Makefile`'s `DBT_CORE_URL`/`DBT_SQLSERVER_URL` overrides let
  anyone point their local clone at a fork/branch without touching this
  repo's history at all.

Where reproducibility genuinely matters — e.g. "this audit was performed
against exactly this commit" in `00-current-state.md` — it's recorded as a
commit hash in prose instead. That doesn't need git to enforce it; if the
audit goes stale as upstream moves, the fix is re-auditing and bumping the
cited hash, not switching to submodules.

## Suggested execution order

For further adapter work, read `01` and `05` first. The port's settled
decisions—quoting, auth surface, TLS parameters, and deferred scope—are
load-bearing. Run the SQL Server-specific test matrix in `04` before widening
the supported release claims.
