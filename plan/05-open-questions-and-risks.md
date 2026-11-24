# 05 — Decisions & Risks

Current on `dbt-core:sqlserver-v2-port` `51db264c1`.

## Settled decisions

1. **Identifiers use double quotes.** `quote_char` is `"`; relation components
   render through the shared `escaped_identifier`, which doubles an embedded
   quote. Live SQL Server 2022 verification confirmed the V2 profile's
   ADBC session has `QUOTED_IDENTIFIER` on; no separate init statement is used.
2. **Authentication includes SQL login and the implemented Entra flows.** An
   omitted `authentication` defaults to `sql`. Windows/trusted-connection auth
   and named instances remain deferred; named instances fail loudly at URI
   parsing rather than risking a connection to the wrong server.
3. **TLS options follow v1's ADBC backend.** `encrypt`, `TrustServerCertificate`
   and connection timeout are supported in `dbt-auth/src/sqlserver/mod.rs`.
   Live SQL Server tests used encryption with certificate trust enabled.
4. **SQL Server catalog reads use three-part `sys.*` views.** The adapter avoids
   `sp_tables`/`sp_columns` pattern matching and current-database limitations.
   Catalog results still depend on SQL Server metadata visibility; minimum
   permissions for least-privilege users have not been established.
5. **The first landing stays experimental and intentionally short of v1 parity.**
   Unsupported configs fail with named errors where implemented; no behavior
   flags are added without a distinct V2 code path.

## Deferred scope

- Windows/integrated auth and named instances.
- Dynamic data masking/denies; they need adapter resolution APIs not present in
  this port.
- User-configured indexes, `full_refresh_build: prebuilt`,
  `table_refresh_method: dml`, scalar-function materializations and table clone.
- Collation-aware case folding.

## Remaining risks

- Live validation covers SQL login on SQL Server 2022 `16.0.4295.3`; Entra has
  unit tests only. Azure SQL Database/Managed Instance and other SQL Server
  versions need separate evidence before claiming support.
- Minimum catalog permissions are unknown; restricted logins may see only a
  subset of `sys.*` objects.
- The view materialization is in the render harness for the three scenarios the
  other adapters use. The skip/rebuild branch inside it — the one place the
  vendored macro diverges from v1 — is covered by the live matrix only: a
  harness test for it would stub `run_query` with a fixed row and assert on
  emitted SQL, which is the property the server shows directly.
- Package compatibility gaps remain: `dbt.date_spine` uses unsupported nested
  `WITH`/`order by 1`, and `dbt_utils.expression_is_true` emits an unaliased
  scalar inside the test wrapper.
- Cancellation reaches a driver that does not cancel. The `mssql` ADBC driver
  1.8.0 exports `AdbcStatementCancel` and returns success without stopping the
  statement (SQL Server 2022 `16.0.4295.3`, `sys.dm_exec_requests` unchanged
  until completion), and the port carries no cancel code of its own, so
  Ctrl-C in Fusion hits the same no-op as dbt-core#15494. v1 has the same gap,
  plus an mssql-python 1.14.0 cursor with no `cancel()`. `KILL <spid>` works but
  needs `ALTER ANY CONNECTION`.
- Upstream PR #15769 is a draft at remote head `51db264c1`, the local branch
  head; it is mergeable and blocked on the required review.
