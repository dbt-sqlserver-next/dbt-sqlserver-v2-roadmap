---
target_repo: dbt-msft/dbt-sqlserver
type: support
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/903
---

# Add dbt Core 1.13 support and test the SQL Server v2 parser

The README says the adapter supports dbt Core 1.12 or newer, and `pyproject.toml` permits `dbt-core>=1.12.0,<2.0`. That range admits 1.13, but the checked-in lock resolves Core 1.12.6 and CI has no explicit Core-version matrix. The install-from-metadata CI jobs may float to a newer compatible Core, but they do not demonstrate that both supported minor lines remain covered.

SQL execution compatibility has not been tested. The 1.13.0b1 notes change defaults for flags introduced in 1.9/1.10, add `--use-v2-parser`, and delay/package-scope unit-test model resolution. The adapter shadows dbt's built-in unit-test fixture and snapshot strategy macros, so these paths need compatibility tests.

## Compatibility gaps

- **Optional v2 parser (handoff verified):** Core 1.13 invokes `dbt-core-experimental-parser parse` when `--use-v2-parser` is set. Its upstream Rust parser does not define SQL Server's `AdapterType`; Core's later Python macro rediscovery cannot add SQL Server dialect support to that Rust parse. The fork implements `AdapterType::SqlServer`. An isolated check with Core 1.13.0b1 and dbt-sqlserver 1.12.2 used the fork's release-built 2.0.8 binary via `--v2-parser "<fork-binary> parse"`. With `--use-v2-parser`, parsing passed for the default and a custom `packages-install-path`; Core loaded the manifest, rediscovered `macro.dbt_sqlserver.sqlserver__create_table_as`, and `dbt compile --no-populate-cache` passed. Core's parser dependency range is `dbt-core-experimental-parser>=2.0.0b1,<3`, which includes 2.0.8. This verifies the explicit command handoff, not installation of the fork wheel/default parser entry point, stable 1.13, SQL execution, or CI coverage.

## Readiness PR spike (not a 1.13 support blocker)

Core 1.13's `catalog_database` override applies to model relations using a catalog-write integration. SQL Server currently has catalog-read macros but no registered `CATALOG_INTEGRATIONS` or catalog-write integration. During readiness PR work, assess whether SQL Server should implement/register one so this override can be used, and test its database-name precedence if pursued. This is a spike to decide, not a requirement for Core 1.13 compatibility or parser support.

## Steps to reproduce

The fork-parser handoff passed with Core 1.13.0b1 using an explicit parser-command override; the default upstream parser rejected `sqlserver` as unknown. In a clean environment, repeat with the first stable 1.13 release and the packaged fork parser, then exercise snapshots, unit tests, and live SQL Server behavior. The current CI configuration does not pin or report a dedicated Core 1.13 lane.

## Expected behavior

Keep CI coverage for both the supported 1.12 line and Core 1.13. Exercise model, snapshot and unit-test behavior and the default-on flags. For `--use-v2-parser`, install the fork-built `dbt-core-experimental-parser` artifact (the upstream Rust parser lacks SQL Server), and cover the manifest handoff, adapter macro rediscovery, and default/custom package-install paths. The command handoff is locally verified; packaged installation, stable 1.13, live SQL behavior, and a CI lane remain unverified. Update dependency/lock configuration and the README's tested-version statement to match results; fix failures with regression coverage before claiming full 1.13 support.

## Environment

- Database: Not connected; parser/manifest and compile checks only.
- Backend: `mssql-python` in the test profile; no connection was opened.
- Authentication, ODBC driver, OS: Not applicable to the source audit.
- Database collation: Not measured.
- Other dbt packages: dbt Core 1.13.0b1; dbt-sqlserver 1.12.2; fork parser binary 2.0.8. The checked-in lock still resolves Core 1.12.6.

## Log excerpt

```text
Running with dbt=1.13.0-b1
dbt-oss 2.0.8
Registered adapter: sqlserver=1.12.2
Found 1 model, 580 macros
```
