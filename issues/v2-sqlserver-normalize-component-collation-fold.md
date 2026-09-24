---
target_repo: dbt-sqlserver-next/dbt-core
type: bug
status: draft
parent: https://github.com/dbt-sqlserver-next/dbt-core/issues/4
related: ../plan/05-open-questions-and-risks.md
---

# `normalize_component` lowercases SQL Server identifiers regardless of collation

`relation_impl.rs` `normalize_component` has no `SqlServer` arm, so SQL Server
takes the catch-all `to_lowercase()`, as Fabric does. Fabric's collation is
always case-insensitive. SQL Server's is set per server, database and column,
and can be `_CS_`.

`base.rs` `semantic_fqn` calls `normalize_component` for every component whose
quote policy is off, so with `quoting: false` the names `MODEL` and `model` get
the same `semantic_fqn`. On a case-sensitive database those are two objects.
With the default policy (all quoted) `normalize_component` isn't called.

v1 has the same fold: `TestCachingUppercaseModel` in
`tests/functional/adapter/dbt/test_caching.py` is skipped because "MODEL is
coereced to model".

## Why it isn't a one-line fix

Nothing in the adapter knows the connection's collation. The options:

- Read it (`DATABASEPROPERTYEX(db, 'Collation')`, or per column from
  `sys.columns.collation_name`) and fold only under `_CI_`. That needs a
  connection where `normalize_component` has none.
- Never fold. That's correct under `_CS_` and makes `MODEL` and `model`
  distinct cache keys under the default `_CI_` collation, where they're one
  object.

`seed_io.rs` `infer_seed_column_name_strategy` and the name predicates in
`metadata/sqlserver` also decide case behavior without knowing the collation,
so the choice should cover all three.
