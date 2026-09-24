---
target_repo: dbt-labs/dbt
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# Unit test `given` schema fetch errors drop the database's message

Running jaffle-shop's unit tests with the SQL Server adapter from #15769, some
unit tests failed before rendering:

```
[error] [ExecutorFailed (dbt1401)]: Remote database error while fetching
schema for unit test upstream relation from `given`
'"TestDB"."dbo"."stg_supplies"'
```

The error carries no driver message. Every other database error in the same
runs includes one (`[mssql] Could not execute query: ...`).

## What's known

- Two builds failed on different relations: the first on `raw.raw_stores` and
  `dbo.stg_supplies`, the second (with #15766 applied) on `dbo.stg_supplies`
  only.
- Both relations had already been built earlier in the same run.
- The message comes from `fetch_schema_for_unit_test_relation`
  (`dbt-tasks-sa` `renderable/unit_test.rs`), in the branch where
  `list_relations_sdf_schemas` returns a per-relation `Err`. It calls
  `into_fs_error(err).with_context(...)`. `into_fs_error` puts the
  `AdapterError` text in `FsError.context`, and `FsError::with_context`
  (`dbt-error` `types.rs`) replaces `context`, so the driver message is
  discarded. The metadata-call error in the same function goes through the same
  pair.
- On SQL Server that per-relation result comes from
  `SqlServerMetadataAdapter::list_relations_schemas_inner`: one catalog query
  (`build_columns_sql`) per relation on a pooled connection, then
  `build_schema_from_columns`.

Not reproduced since, and not reduced to a single relation or query.

## Asks

1. Keep the `AdapterError` text at these two call sites, e.g. by folding it
   into the context string. Without it, the failure can't be told apart from a
   timing problem, a query error or a schema-building error.
2. With that in place, the root cause can be re-run and filed separately.
