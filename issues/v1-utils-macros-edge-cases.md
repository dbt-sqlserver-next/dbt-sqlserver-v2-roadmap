---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: draft
related: ../plan/04-testing-and-validation.md
---

# `date_trunc` drops the time below day, `split_part` fails on `&`, `len` drops trailing spaces and `listagg` ignores `limit_num`

Measured on SQL Server 2022 (16.0.4295.3, Linux), mssql-python backend, against v1.12.0.

`tests/functional/adapter/dbt/test_utils.py` runs the shared dbt suites for these macros, which cover `day` and `month` for `date_trunc` and, for `listagg`, an expected file that matches the current output.

## `date_trunc` returns a `DATE` for every part

`sqlserver__date_trunc` is `CAST(DATEADD(p, DATEDIFF(p, 0, x), 0) AS DATE)`.

| Part | Result for `2024-05-06 13:45:10.123` |
|---|---|
| `year` … `day` | correct |
| `hour` | `2024-05-06` (should be `2024-05-06 13:00`) |
| `minute` | `2024-05-06` (should be `2024-05-06 13:45`) |
| `second`, `millisecond` | Msg 535, `datediff` overflow: seconds since 1900 exceed `int` |

`DATEDIFF_BIG` does not help; `DATEADD` takes an `int` number. Anchoring at the value's own day fits every sub-day part and returns the truncated value:

```sql
DATEADD(second, DATEDIFF(second, CAST(CAST(x AS date) AS datetime2), x), CAST(CAST(x AS date) AS datetime2))
-- 2024-05-06 13:45:10
```

Same expression checked for `hour`, `minute`, `second` and `millisecond`.

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
select {{ dbt.date_trunc('hour', "cast('2024-05-06 13:45:10' as datetime2)") }}   -- 2024-05-06
select {{ dbt.date_trunc('second', "cast('2024-05-06 13:45:10' as datetime2)") }} -- Msg 535
select {{ dbt.split_part("'a&b,c'", "','", 1) }}                                   -- XML parsing error
select {{ dbt.length("'a  '") }}                                                   -- 1
select {{ listagg('v', "','", 'order by v', 2) }} from (values ('a'),('b'),('c')) t(v)  -- a,b,c
```

## Environment

- Database: SQL Server 2022 (16.0.4295.3), Linux, Developer Edition
- Backend: mssql-python
- dbt-sqlserver: v1.12.0
