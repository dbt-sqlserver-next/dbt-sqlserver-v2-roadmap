---
target_repo: dbt-msft/dbt-sqlserver
type: support
status: draft
---

# Validate dbt Core 1.13 support and catalog/parser features

The README says the adapter supports dbt Core 1.12 or newer, and `pyproject.toml` permits `dbt-core>=1.12.0,<2.0`. That range admits 1.13, but the checked-in lock resolves Core 1.12.6 and CI has no explicit Core-version matrix. The install-from-metadata CI jobs may float to a newer compatible Core, but they do not demonstrate that both supported minor lines remain covered.

No 1.13 runtime incompatibility has been reproduced. The 1.13.0b1 notes change defaults for flags introduced in 1.9/1.10, add `--use-v2-parser`, and delay/package-scope unit-test model resolution. The adapter shadows dbt's built-in unit-test fixture and snapshot strategy macros, so these paths need compatibility tests.

## Feature gaps

- **Catalog database override:** Core 1.13 adds `catalog_database` to `CatalogWriteIntegrationConfig`; the default `generate_database_name` uses it for model relations associated with a configured catalog integration. SQL Server implements catalog reads (`sqlserver__get_catalog`, `sqlserver__get_catalog_relations`) but registers no `CATALOG_INTEGRATIONS` and has no catalog-write integration. Consequently SQL Server cannot currently use this override. Supporting it requires implementing/registering a SQL Server catalog-write integration, then testing that `catalog_database` takes precedence for those model relations without changing ordinary model database naming.
- **Optional v2 parser:** Core 1.13 invokes `dbt-core-experimental-parser parse` when `--use-v2-parser` is set. `dbt-core` already depends on `dbt-core-experimental-parser>=2.0.0b1,<3`; it is a separate Rust parser executable, not the `dbt-sqlserver-next/dbt-core` fork. Core re-discovers macros from the installed Python adapter after parsing. No adapter dependency override is needed; `--v2-parser` changes the executable command, not the dependency. The adapter has no explicit CI coverage for this mode.

## Steps to reproduce

There is no known failing runtime case. In a clean environment, install this adapter with `dbt-core==1.13.0b1` (then repeat with the first stable 1.13 release), run `dbt parse --use-v2-parser`, and run the adapter's unit and SQL Server functional suites. The current CI configuration does not pin or report a dedicated Core 1.13 lane. SQL Server currently has no registered catalog-write integration to configure with `catalog_database`.

## Expected behavior

Keep CI coverage for both the supported 1.12 line and Core 1.13. Exercise model, snapshot and unit-test behavior, the default-on flags, and adapter macro dispatch with `--use-v2-parser`; use Core's declared `dbt-core-experimental-parser` dependency rather than overriding it with the V2 port. Implement and register SQL Server catalog-write integration support, including the 1.13 `catalog_database` precedence, with functional tests against SQL Server. Update dependency/lock configuration and the README's tested-version statement to match results; fix failures with regression coverage before claiming support.

## Environment

- dbt-sqlserver checkout: `chore/1.12.2rc1`, package version `1.12.2rc1`
- dbt Core: 1.13.0b1 pre-release (release page checked 2026-11-02; current stable 1.x: 1.12.6)
- Database/backend: SQL Server functional tests required; not run for this audit

`dbt --version` from the checked-in lock resolves Core 1.12.6; no 1.13 adapter or `--use-v2-parser` run has been recorded.

## Log excerpt

Not applicable: this request is for explicit compatibility testing; no runtime failure is known.
