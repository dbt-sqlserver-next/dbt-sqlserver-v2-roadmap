---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# [v2 Bug] Unit test `given` schema fetch errors drop the database's message

### Is this a new bug in dbt v2.x compared to the latest version of dbt 1.x?

- [X] I believe this is a new bug in dbt v2.x
- [X] I have searched the existing issues and could not find a duplicate

### Current Behavior

When fetching a `given` relation's schema fails, the error carries no driver message. Every other database error in the same runs includes one (`[mssql] Could not execute query: ...`).

The message comes from `fetch_schema_for_unit_test_relation` (`dbt-tasks-sa` `renderable/unit_test.rs`), in the branch where `list_relations_sdf_schemas` returns a per-relation `Err`. It calls `into_fs_error(err).with_context(...)`. `into_fs_error` puts the `AdapterError` text in `FsError.context`, and `FsError::with_context` (`dbt-error` `types.rs`) replaces `context`, so the driver message is discarded. The metadata-call error in the same function goes through the same pair.

### Expected Behavior

The error keeps the `AdapterError` text, so the failure can be told apart from a timing problem, a query error or a schema-building error.

### Steps To Reproduce

Not reduced to a single relation or query. Running jaffle-shop's unit tests with the SQL Server adapter from #15769, some unit tests failed before rendering:

- Two builds failed on different relations: the first on `raw.raw_stores` and `dbo.stg_supplies`, the second (with #15766 applied) on `dbo.stg_supplies` only.
- Both relations had already been built earlier in the same run.
- Not reproduced since.

On SQL Server the per-relation result comes from `SqlServerMetadataAdapter::list_relations_schemas_inner`: one catalog query (`build_columns_sql`) per relation on a pooled connection, then `build_schema_from_columns`.

### Relevant log output

```shell
[error] [ExecutorFailed (dbt1401)]: Remote database error while fetching
schema for unit test upstream relation from `given`
'"TestDB"."dbo"."stg_supplies"'
```

### Environment

- OS: Linux
- CPU: x86
- dbt distribution and version: `dbt-labs/dbt` `main` source with the SQL Server adapter from #15769; SQL Server 2022

### Which database adapter are you using?

other (describe in Additional Context)

### Is this a discrepancy vs. dbt 1.x?

- [ ] Yes — this works in dbt 1.x but not in dbt v2.x

### Additional Context

Adapter: SQL Server (#15769). The dropped message is in engine code shared by all adapters.

Keep the `AdapterError` text at these two call sites, e.g. by folding it into the context string. With that in place, the underlying failure can be re-run and filed separately.
