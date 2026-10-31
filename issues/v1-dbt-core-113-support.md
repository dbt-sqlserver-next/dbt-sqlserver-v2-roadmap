---
target_repo: dbt-msft/dbt-sqlserver
type: support
status: draft
---

# Add explicit dbt Core 1.13 compatibility coverage

The README says the adapter supports dbt Core 1.12 or newer, and `pyproject.toml` permits `dbt-core>=1.12.0,<2.0`. That range admits 1.13, but the checked-in lock resolves Core 1.12.6 and CI has no explicit Core-version matrix. The install-from-metadata CI jobs may float to a newer compatible Core, but they do not demonstrate that both supported minor lines remain covered.

No 1.13 incompatibility has been reproduced. The 1.13.0b1 release notes do introduce compatibility-sensitive behavior worth checking: defaults for flags introduced in 1.9/1.10 change to true; `--use-v2-parser` is available and re-parses installed adapter macros; and unit-test model resolution is delayed and package-scoped. The adapter shadows dbt's built-in unit-test fixture and snapshot strategy macros, and overrides adapter/connection contracts, so a successful install alone is insufficient evidence. The `dbt-adapters` minimum is unchanged from 1.12.

## Steps to reproduce

There is no known failing runtime case. In a clean environment, install this adapter with `dbt-core==1.13.0b1` (then repeat with the first stable 1.13 release) and run the adapter's unit and SQL Server functional suites. The current CI configuration does not pin or report a dedicated Core 1.13 lane.

## Expected behavior

Declare 1.13 support based on passing tests, not only the broad dependency range. Keep at least one CI lane on the supported 1.12 line and add a 1.13 lane; exercise standard model/snapshot/unit-test behavior and the adapter's overridden macros against SQL Server. Check the 1.13 default-on flags and decide whether the optional `--use-v2-parser` path is included in the support promise; if included, test adapter macro rediscovery. Update the supported-version documentation and dependency/lock configuration to match the verified range. Fix any failures with regression coverage before claiming support.

The new `catalog_database` field for catalog write integrations is not currently exposed by this adapter; treat it as a separate feature decision, not an automatic 1.13 compatibility blocker.

## Environment

- dbt-sqlserver checkout: `chore/1.12.2rc1`, package version `1.12.2rc1`
- dbt Core: 1.13.0b1 pre-release (current stable 1.x checked: 1.12.6)
- Database/backend: SQL Server functional tests required; not run for this audit

`dbt --version` from the checked-in lock resolves Core 1.12.6; no 1.13 adapter run has been recorded.

## Log excerpt

Not applicable: this request is for explicit compatibility testing; no runtime failure is known.
