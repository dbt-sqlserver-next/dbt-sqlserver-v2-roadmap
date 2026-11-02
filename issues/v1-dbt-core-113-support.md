---
target_repo: dbt-msft/dbt-sqlserver
type: support
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/903
---

# Add dbt Core 1.13 support and test the SQL Server v2 parser

The README says the adapter supports dbt Core 1.12 or newer, and `pyproject.toml` permits `dbt-core>=1.12.0,<2.0`. That range admits 1.13, but the checked-in lock resolves Core 1.12.6 and CI has no explicit Core-version matrix. The install-from-metadata CI jobs may float to a newer compatible Core, but they do not demonstrate that both supported minor lines remain covered.

No 1.13 runtime incompatibility has been reproduced. The 1.13.0b1 notes change defaults for flags introduced in 1.9/1.10, add `--use-v2-parser`, and delay/package-scope unit-test model resolution. The adapter shadows dbt's built-in unit-test fixture and snapshot strategy macros, so these paths need compatibility tests.

## Compatibility gaps

- **Optional v2 parser:** Core 1.13 invokes `dbt-core-experimental-parser parse` when `--use-v2-parser` is set. Its upstream Rust parser does not define a SQL Server `AdapterType`; Core re-discovers macros from the installed Python adapter after parsing, but that cannot add SQL Server dialect support to the Rust parse. The SQL Server fork does implement `AdapterType::SqlServer` and its release workflow packages the Rust `dbt` binary as the `dbt-core-experimental-parser` distribution. The parser test lane therefore needs to replace Core's upstream `dbt-core-experimental-parser` dependency with the fork-built package (or explicitly configure `--v2-parser` to the fork executable); it does not need to replace `dbt-core` itself. Core's range is `dbt-core-experimental-parser>=2.0.0b1,<3`, so verify the fork package version satisfies it. Test both default and custom `--packages-install-path` values through the handoff.

## Readiness PR spike (not a 1.13 support blocker)

Core 1.13's `catalog_database` override applies to model relations using a catalog-write integration. SQL Server currently has catalog-read macros but no registered `CATALOG_INTEGRATIONS` or catalog-write integration. During readiness PR work, assess whether SQL Server should implement/register one so this override can be used, and test its database-name precedence if pursued. This is a spike to decide, not a requirement for Core 1.13 compatibility or parser support.

## Steps to reproduce

There is no known failing runtime case. In a clean environment, install this adapter with `dbt-core==1.13.0b1` (then repeat with the first stable 1.13 release). Run the normal Python-parser path, then run `dbt parse --use-v2-parser` with `dbt-core-experimental-parser` built from the SQL Server fork; run the adapter's unit and SQL Server functional suites. The current CI configuration does not pin or report a dedicated Core 1.13 lane.

## Expected behavior

Keep CI coverage for both the supported 1.12 line and Core 1.13. Exercise model, snapshot and unit-test behavior and the default-on flags. For `--use-v2-parser`, install the fork-built `dbt-core-experimental-parser` artifact (the upstream Rust parser lacks SQL Server), and verify Core's manifest handoff, adapter macro rediscovery, and default/custom package-install paths. Update dependency/lock configuration and the README's tested-version statement to match results; fix failures with regression coverage before claiming support.

## Environment

- Database: Not run; this is a compatibility-coverage request.
- Backend: Not run.
- Authentication, ODBC driver, OS: Not applicable to the source audit.
- Database collation: Not measured.
- Other dbt packages: dbt Core 1.13.0b1 is the prerelease target; the checked-in lock resolves 1.12.6.

`dbt --version`:
```text
No 1.13 adapter run has been recorded. The checked-in lock resolves dbt-core 1.12.6.
```

## Log excerpt

```text
Not applicable: no runtime failure has been reproduced.
```
