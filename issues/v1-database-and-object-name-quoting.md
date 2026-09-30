---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: v1-use-database-deletes-embedded-quote.md
---

# Names containing `;`, `'`, `-` or `&` break the connection string and five macros

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

An unusual name reaches a connection string or generated SQL unquoted. Related: [v1-use-database-deletes-embedded-quote](v1-use-database-deletes-embedded-quote.md), [v1-use-database-state-vs-unqualified-catalog-reads](v1-use-database-state-vs-unqualified-catalog-reads.md), [v1-varchar-name-literals-non-codepage](v1-varchar-name-literals-non-codepage.md).

## A database name containing `;` breaks the connection string

`build_common_connection_string_parts` writes `Database={credentials.database}` and `SERVER=<host>` as-is, while `UID`, `PWD`, `client_id` and `client_secret` go through `format_connection_string_value`. A `;` in the value ends the keyword early. With `create database [qw;db]` and `database: qw;db` in the profile, the string is:

```
SERVER=127.0.0.1,1433;Database=qw;db;UID=SA;PWD=***;encrypt=No;TrustServerCertificate=Yes
```

and the connection fails with `Connection string parsing failed: Incomplete specification: keyword 'db' has no value (missing '=')`.

`Database={qw;db}` connects and `db_name()` returns `qw;db`, so passing the database (and the host argument) through `format_connection_string_value` fixes it. Measured on mssql-python; the pyodbc string is built the same way.

## `drop_schema` doesn't quote the schema

`sqlserver__drop_schema` runs `EXEC('DROP SCHEMA IF EXISTS <schema>')` with the name as-is, while `create_schema` quotes it. For a schema named `qw-x`, created with `create schema [qw-x]`, the drop fails with `Incorrect syntax near '-'.` Quoting through `adapter.quote` and `escape_single_quotes` matches `create_schema_if_not_exists`.

## `get_provision_sql` doesn't escape the principal

`get_provision_sql` (`auto_provision_aad_principals`) tests `where name = '{{ grantee }}'` without escaping. A principal named `O'Brien` fails with `Incorrect syntax near 'Brien'.` before the `create user` runs. `escape_single_quotes(grantee)` matches the rest of the file.

## `get_tables_by_pattern_sql` doesn't quote the database

`sqlserver__get_tables_by_pattern_sql` renders `{{ database }}.INFORMATION_SCHEMA.TABLES` bare. For a database named `qw-db`, `select … from qw-db.INFORMATION_SCHEMA.TABLES` fails with `Incorrect syntax near '-'.` and `"qw-db".INFORMATION_SCHEMA.TABLES` works. `adapter.quote(database)` fixes it.

## `drop_*` macros entity-escape names

`drop_xml_indexes`, `drop_spatial_indexes`, `drop_fk_constraints`, `drop_pk_constraints` and `drop_all_indexes_on_table` build their DDL with `select ... for xml path('')` and assign it to `nvarchar(max)`. `FOR XML PATH` escapes `&`, `<` and `>`, so an index named `ix&b` produces:

```
DROP INDEX [ix&amp;b] ON dbo.qw_t;
```

which names an index that doesn't exist. `for xml path(''), type).value('.', 'nvarchar(max)')` returns the text as built.

## Steps to reproduce

```sql
create table dbo.qw_t (a int, b int);
create index [ix&b] on dbo.qw_t (a);
declare @s nvarchar(max);
select @s = (select 'DROP INDEX ' + quotename(i.[name]) + ' ON dbo.qw_t; '
             from sys.indexes i where i.object_id = object_id('dbo.qw_t') and i.name is not null
             for xml path(''));
select @s;  -- DROP INDEX [ix&amp;b] ON dbo.qw_t;
```

For the connection string, set `database: qw;db` after `create database [qw;db]`. For the others, use a schema `qw-x`, a principal `O'Brien` with `auto_provision_aad_principals`, and a database `qw-db`.

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
