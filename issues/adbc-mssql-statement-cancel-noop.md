---
target_repo: adbc-drivers/mssql
type: bug
status: open
url: https://github.com/adbc-drivers/mssql/issues/19
related: ../plan/05-open-questions-and-risks.md
---

# AdbcStatementCancel returns success without stopping the running query

**Driver Version:** 1.8.0 (`dbc install mssql`, `mssql_linux_amd64_v1.8.0`), with `adbc-driver-manager` 1.12.0

**Microsoft SQL Server Version:** 2022 `16.0.4295.3` (Linux container)

## What happened?

`AdbcStatementCancel` returns success while the query keeps executing, and the
blocked `execute_query` returns normally instead of `ADBC_STATUS_CANCELLED`.

`adbc.h` documents the contract (`AdbcStatementCancel`, API revision 1.1.0):
"Calling this function should make the other functions return
ADBC_STATUS_CANCELLED … (It is not guaranteed to, for instance, the result set
may be buffered in memory already.)" The caveat doesn't apply: nothing was
buffered, and the request was still in `sys.dm_exec_requests` 9 seconds after
the cancel returned.

The call reaches the driver — with no active query it raises `INVALID_STATE:
[mssql] AdbcStatementCancel: no active query to cancel`, and the 1.8.0 binary
exports `AdbcStatementCancel` and `AdbcConnectionCancel` — so this is not the
driver-manager stub. Statement-level and connection-level cancels behave the
same.

Measured, `WAITFOR DELAY '00:00:10'` started on a worker thread, cancel at
t=1.0s:

| t (s) | `sys.dm_exec_requests` for the session | Client |
|---|---|---|
| 1.00 | `WAITFOR`, elapsed 1001 ms | `adbc_cancel()` returns OK |
| 2.00 | `WAITFOR`, elapsed 2004 ms | |
| 3.01 | `WAITFOR`, elapsed 3007 ms | |
| 10.00 | gone | `execute_query` returns normally |

Impact: a cancelled query keeps running server-side on the connection's session.
dbt core 1.x (`cancel_open` → `cursor.adbc_cancel()`), and Fusion's
`cancel_all_tracked_statements` → `ConnectionCancel`/`StatementCancel`, both get
success and no cancellation. That is the failure dbt-labs/dbt-core#15494 fixed
for Redshift at the driver level; on SQL Server it would leave the statement
running after the CLI reports the interrupt.

## Stack Trace

None: the cancel reports success.

## How can we reproduce the bug?

```python
import threading, time
from adbc_driver_manager import dbapi

URI = ("sqlserver://SA:YourPassword@localhost:1433?database=master"
       "&encrypt=true&TrustServerCertificate=true")

conn = dbapi.connect(driver="mssql", db_kwargs={"uri": URI}, autocommit=True)
monitor = dbapi.connect(driver="mssql", db_kwargs={"uri": URI}, autocommit=True)
cur = conn.cursor()
cur.execute("SELECT @@SPID")
spid = cur.fetchone()[0]

def run():
    cur.execute("WAITFOR DELAY '00:00:10'; SELECT 1")

t = threading.Thread(target=run)
t.start()
time.sleep(1)

cur.adbc_cancel()  # returns OK

def still_running():
    probe = monitor.cursor()
    probe.execute(
        f"SELECT command FROM sys.dm_exec_requests WHERE session_id = {spid}"
    )
    row = probe.fetchone()
    probe.close()
    return row

print(still_running())  # ('WAITFOR',) expected, () after a working cancel
t.join()
print(still_running())  # (), reached after the full 10s
```

`Connection.adbc_cancel()` and `Statement.cancel()` show the same behaviour, and
`cursor.adbc_cancel()` after `execute_query` returns raises `INVALID_STATE` as
documented.

## Environment/Setup

- Driver 1.8.0 on Linux x86_64, `adbc-driver-manager` 1.12.0 (Python 3.11)
- SQL Server 2022 `16.0.4295.3`, SQL login (sysadmin), encrypted connection
- Reported from dbt-sqlserver's `backend: adbc`; the same call is made by dbt
  Fusion.
