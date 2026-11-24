---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `split_part` fails on `&` and cuts at 128 characters, `len` drops trailing spaces, `listagg` ignores `limit_num`

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.3.

`tests/functional/adapter/dbt/test_utils.py` runs the shared dbt suites for these macros, and for `listagg` its expected file matches the current output.

## `split_part` fails or truncates

`sqlserver__split_part` builds XML from the string and reads the part with `.value('(/X)[n]', 'VARCHAR(128)')`.

- A value containing `&` or `<` fails: `select cast('<X>'+replace('a&b,c', ',', '</X><X>')+'</X>' as xml)` raises `XML parsing: line 1, character 7, semicolon expected`.
- A part longer than 128 characters is cut to 128 with no error (a 300-character part returns 128).

`VARCHAR(MAX)` removes the truncation and returns `varchar` as before. Escaping `&`, `<` and `>` in the input removes the parse error.

## `len` drops trailing spaces

`sqlserver__length` is `len()`, which ignores trailing spaces: `len('a  ')` is 1, where `length` elsewhere counts 3. `len(<expr> + 'x') - 1` returns 3 for `varchar`, `nvarchar` and `varchar(max)` inputs, keeps `NULL` as `NULL`, and returns `int` (`bigint` for a `max` input, as `len` does). A non-string argument then needs a cast (`len(1 + 'x')` raises a conversion error), so the README states that `length` takes a string expression.

## `listagg` ignores `limit_num`

`sqlserver__listagg` never reads `limit_num`. `listagg('v', "','", 'order by v', 2)` over `a`, `b`, `c` returns `a,b,c`. The shared `TestListagg` passes because the adapter's `data_listagg_output.csv` expects the unbounded `c_|_b_|_a` for its `limit_num=2` case.

T-SQL has no bounded aggregate that fits inside an expression, so the macro can't honour the limit. Raising a compiler error when `limit_num` is set turns a wrong result into a visible one; the test drops its `limit_num` case, and the README states that `limit_num` isn't supported.

## Steps to reproduce

```sql
select {{ dbt.split_part("'a&b,c'", "','", 1) }}                                   -- XML parsing error
select {{ dbt.length("'a  '") }}                                                   -- 1
select {{ listagg('v', "','", 'order by v', 2) }} from (values ('a'),('b'),('c')) t(v)  -- a,b,c
```

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.3
