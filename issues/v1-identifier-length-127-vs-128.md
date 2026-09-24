---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/02-implementation-steps.md
---

# `MAX_CHARACTERS_IN_IDENTIFIER` is 127, Redshift's limit; SQL Server accepts 128

`dbt/adapters/sqlserver/relation_configs/policies.py` sets
`MAX_CHARACTERS_IN_IDENTIFIER = 127`. `SQLServerRelation.__post_init__` raises
above it, and `relation_max_name_length()` returns it to Jinja. SQL Server
identifiers are `sysname`, i.e. `nvarchar(128)`.

Measured on SQL Server 2022 (16.0.4265.3, Linux), against v1.12.0rc4.

## The server accepts 128

- `SELECT max_length/2 FROM sys.types WHERE name = 'sysname'` returns 128.
- 128-character table, column and schema names are created, bracket- or
  double-quoted. 129 fails with `Msg 103: ... Maximum length is 128.`
- Local `#` temp tables cap at 116 (`Msg 193`), but
  `sqlserver__make_temp_relation` builds a regular table with a `__dbt_temp`
  suffix, so that limit doesn't apply here.

## The adapter rejects 128

```python
>>> SQLServerRelation.create(database="d", schema="s", identifier="a"*127, type="table")
accepted
>>> SQLServerRelation.create(database="d", schema="s", identifier="a"*128, type="table")
DbtRuntimeError: Relation name 'aaa…' is longer than 127 characters
```

Exactly one length is affected, and it fails at parse time. dbt's truncation
logic and packages such as `dbt_utils` read `relation_max_name_length()`, so
generated names come out one character shorter too. No test covers the
boundary.

## Where 127 came from

It's Redshift's value, copied with the scaffolding:

- `dbt-redshift/src/dbt/adapters/redshift/relation_configs/policies.py` in
  dbt-labs/dbt-adapters has the same path, constant and value.
- The constant, the raise and `relation_max_name_length()` all arrived in
  `a204adb` ("Updated to 1.8 and rebuilt test suite"), which created
  `relation_configs/` wholesale.
- The check in `SQLServerRelation.__post_init__` is still commented
  `# Check for length of Redshift table/view names.`

## Fix

1. Set `MAX_CHARACTERS_IN_IDENTIFIER = 128`, citing `sysname`.
2. Replace the Redshift comment in `SQLServerRelation.__post_init__`.
3. Add a boundary test (128 accepted, 129 rejected), next to
   `tests/unit/adapters/mssql/test_quote.py`.

Separately: a model name over 118 characters plus `__dbt_temp` exceeds 128, and
the resulting error names the temp relation rather than the model. That's worth
its own issue.

The v2 adapter uses 128 (`dbt-adapter-sql` `ident.rs`, `SqlServer` arm), so
until this lands a 128-character name fails under v1 and builds under v2.

## References

- https://learn.microsoft.com/sql/relational-databases/databases/database-identifiers
