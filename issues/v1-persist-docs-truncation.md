---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `persist_docs` cuts descriptions at 3750 characters without a warning

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

`sqlserver__alter_relation_comment` and `sqlserver__alter_column_comment` declare the value as `nvarchar(3750)`, which is what an extended property (`sql_variant`, 7,500 bytes) holds. A 5,000-character model description and a 5,000-character column description are both stored as 3,750 characters, with no warning.

A warning naming the relation (and column) when the description exceeds 3750 characters makes the cut visible.

## Steps to reproduce

Set `persist_docs: {relation: true, columns: true}` on a table model whose description is 5,000 characters, then read `sys.extended_properties`: both values have length 3750.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
