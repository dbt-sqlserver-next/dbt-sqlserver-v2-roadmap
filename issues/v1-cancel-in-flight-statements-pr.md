---
target_repo: dbt-msft/dbt-sqlserver
type: pr-draft
status: closed
url: https://github.com/dbt-msft/dbt-sqlserver/pull/923
base: master
head: axellpadilla:fix/922-cancel-in-flight-statements
closes: https://github.com/dbt-msft/dbt-sqlserver/issues/922
merged: 2026-11-24
related: adbc-mssql-statement-cancel-noop.md
---

# Draft: PR body for `fix/922-cancel-in-flight-statements` → `dbt-msft/dbt-sqlserver:master`

Merged 2026-11-24 as `fix(cancel): stop in-flight statements on adbc and
mssql-python`, closing #922.

## Branch state

- Branched off `master` `dacb4b8f`, three commits: `d14c4bbc` (ADBC),
  `059b3d95` (mssql-python), `98ed6505` (functional test).
- Fragments: `changes/922.bugfix.md`, `changes/+mssql-python-cursor-cancel.bugfix.md`.
  `CHANGELOG.md` untouched.
- Docs: one row plus a caveat paragraph in `docs/adbc_backend.md`.

## Notes for filing

- Title below, or squash into one commit under it; the commits separate the
  driver-facing change from the private-handle bridge and the functional test.
- The body says `Closes #922`. Its acceptance criterion "confirm the SQL Server
  statement stops" cannot pass on the ADBC driver 1.8.0; the ADBC bullet states
  that in the body, and the functional test xfails on that backend, so a
  reviewer sees the limit before closing the issue.
- The driver report is filed as
  [adbc-drivers/mssql#19](https://github.com/adbc-drivers/mssql/issues/19); the
  PR body does not link it.

## Draft PR

**Title:** `fix(cancel): stop in-flight statements on adbc and mssql-python`

`cancel()` skipped every backend except pyodbc: ADBC's DB-API cursor exposes `adbc_cancel()` instead of `cancel()`, and mssql-python 1.14.0 exposes no public `cancel()` at all, so `cancel_open()` found nothing callable and returned while the statement kept running. This probes the three shapes in order and leaves the pyodbc path untouched.

- **Probe order:** `cancel()`, then `adbc_cancel()`, then mssql-python's statement handle. Checked against the installed packages: pyodbc 5.3.0 has `Cursor.cancel`; `dir(adbc_driver_manager.dbapi.Cursor)` is `adbc_cancel, close`; mssql-python 1.14.0 has neither.
- **Statement-level ADBC cancel, not `Connection.adbc_cancel()`:** the connection-level call cancels the session that dbt's sibling connections share. It is also the call Fusion's `cancel_all_tracked_statements` makes.
- **mssql-python through the handle:** `Cursor.hstmt._cancel` is the SQLCancel binding the driver itself calls to unblock a fetch on another thread (`mssql_python/cursor.py`, `arrow_reader.close`). There is no public method yet (microsoft/mssql-python#424). Gated on the cursor's module, so no other driver's handle is reached into, and feature-detected, so a public `cancel()` takes over when it ships.
- **ADBC stays a no-op, and the docs say so:** the `mssql` driver 1.8.0 exports `AdbcStatementCancel` and returns success without stopping the statement (`sys.dm_exec_requests` still shows it at t=3.0s of a 10s `WAITFOR`). Closing the connection from the cancelling thread segfaults `adbc-driver-manager` 1.12.0, and `KILL <spid>` needs `ALTER ANY CONNECTION`, which ordinary logins do not hold (error 6102). Hence a doc caveat instead of a workaround.

Closes #922.

## Verification

SQL Server 2022 `16.0.4295.3` from the repo's docker compose, dbt-core 1.12.6, dbt-adapters 1.24.5.

- `uv run pytest tests/unit`: 673 passed, 5 skipped. New tests: `test_cancel_falls_back_to_adbc_cancel`, `test_cancel_falls_back_to_mssql_python_statement_handle`, `test_cancel_ignores_statement_handle_of_other_drivers`.
- `SQLSERVER_TEST_BACKEND=adbc uv run pytest tests/functional/adapter/mssql/test_limit_none.py`: 2 passed.
- `uv run pytest tests/functional/adapter/mssql/test_cancel_in_flight_statement.py`: passes on `mssql-python` (1.07s) and `pyodbc` (0.90s); on `adbc` it is a strict xfail (20.78s), so it turns into a failure as soon as the driver starts cancelling. The test starts the statement on a worker thread, cancels through `cancel_open_connections()` — the path dbt-core takes on Ctrl-C — and asserts the request leaves `sys.dm_exec_requests` within 5s of a 20s `WAITFOR`.
- `pre-commit run --files <changed files>`: `ruff check`, `ruff format`, `ty` pass.
- Driver level, `WAITFOR DELAY '00:00:10'` with the cancel at t=1.0s: pyodbc `cursor.cancel()` and mssql-python `hstmt._cancel()` both clear the server request by t=1.26s (`HY008` / `Operation canceled`) and leave the connection usable; `adbc_cancel()` leaves it running to t=10.0s.
- `dbt run`, two models with `WAITFOR DELAY '00:00:20'` pre-hooks, SIGINT at t=10s: `backend: mssql-python` ends both requests and exits in 0.88s (previously it waited the 20s out); `backend: adbc` still runs both to completion and exits after them.

ADBC is the one backend the suite cannot assert: the strict xfail records the driver no-op and starts failing when a driver cancels.
