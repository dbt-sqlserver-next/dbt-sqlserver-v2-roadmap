# 05 — Decisions & Risks

Current on `dbt-core:sqlserver-v2-port` `bad853106`.

## Settled decisions

1. **Identifiers use double quotes.** `quote_char` is `"`; `quoted()` doubles
   embedded quotes. Live SQL Server 2022 verification confirmed the V2 profile's
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
- v1's view catalog plan-reuse optimization (`916fc235`) is a possible
  follow-up, not part of this RC refresh.

## Remaining risks

- Live validation covers SQL login on SQL Server 2022 `16.0.4295.3`; Entra has
  unit tests only. Azure SQL Database/Managed Instance and other SQL Server
  versions need separate evidence before claiming support.
- Minimum catalog permissions are unknown; restricted logins may see only a
  subset of `sys.*` objects.
- Package compatibility gaps remain: `dbt.date_spine` uses unsupported nested
  `WITH`/`order by 1`, and `dbt_utils.expression_is_true` emits an unaliased
  scalar inside the test wrapper.
- Upstream PR #15769 remains draft at remote head `4d70a106`; the local refreshed
  branch has not been pushed and upstream CI has not run on it.
