# Issues

Drafts of issues and PRs filed on other repos, reviewed here first. Each file's
frontmatter carries `target_repo`, `status` and, once filed, the live `url`.
`status` is one of:

- `draft`: not filed yet
- `open`: filed, and still open on the target repo
- `closed`: resolved; the file stays as the filed record

A filed draft mirrors the live body. When they differ, the draft holds the
intended text and the live issue needs syncing by hand (see AGENTS.md,
"Outward-facing actions").

Files stay flat and keep their names because live issues and `plan/` link to
them by path. Update this index in the same change as a file's `status`.

## Templates

A draft follows its target repo's issue template: title prefix and section
headings. `gh issue create --body-file` bypasses the web form, so the draft has
to carry both itself, and form labels are not applied.

| Target | Kind | Template | Title | Sections |
|---|---|---|---|---|
| dbt-labs/dbt | 1.x bug | [bug-report.yml](https://github.com/dbt-labs/dbt/blob/main/.github/ISSUE_TEMPLATE/bug-report.yml) | `[1.x Bug] …` | new-bug checkboxes, Current Behavior, Expected Behavior, Steps To Reproduce, Relevant log output, Environment, adapter, Additional Context |
| dbt-labs/dbt | v2 bug | [bug-report-v2.yml](https://github.com/dbt-labs/dbt/blob/main/.github/ISSUE_TEMPLATE/bug-report-v2.yml) | `[v2 Bug] …` | as 1.x, plus "Is this a discrepancy vs. dbt 1.x?" |
| dbt-labs/dbt | regression | [regression-report.yml](https://github.com/dbt-labs/dbt/blob/main/.github/ISSUE_TEMPLATE/regression-report.yml) | `[Regression] …` | as 1.x bug, plus versions spanned; Expected/Previous Behavior |
| dbt-labs/dbt | feature | [feature-request.yml](https://github.com/dbt-labs/dbt/blob/main/.github/ISSUE_TEMPLATE/feature-request.yml) | `[Feature] …` | dbt version, feature, alternatives, who benefits, contributing |
| dbt-labs/dbt-adapters | bug (1.x shared macros and adapters) | [bug-report.yml](https://github.com/dbt-labs/dbt-adapters/blob/main/.github/ISSUE_TEMPLATE/bug-report.yml) | `[Bug] …` | new-bug checkboxes, affected packages, Current, Expected, Steps, log, Environment, Additional Context |
| dbt-labs/dbt-utils | bug | [bug_report.md](https://github.com/dbt-labs/dbt-utils/blob/main/.github/ISSUE_TEMPLATE/bug_report.md) | no prefix | Describe the bug, Steps to reproduce, Expected results, Actual results, log output, System information |
| dbt-msft/dbt-sqlserver | bug | [bug_report.md](https://github.com/dbt-msft/dbt-sqlserver/blob/master/.github/ISSUE_TEMPLATE/bug_report.md) | no prefix | description, Steps to reproduce, Environment (database, backend, auth/driver/OS, collation), `dbt --version`, Log excerpt |
| dbt-sqlserver-next/dbt-core | any | same files as dbt-labs/dbt | as dbt-labs/dbt | as dbt-labs/dbt |

dbt-labs/dbt and dbt-adapters disable blank issues. dbt-labs/dbt's chooser
sends 1.x adapter issues to dbt-adapters, which also ships the 1.x
`global_project` macros.

## Not filed yet

| Draft | Target | What it is | Before filing |
|---|---|---|---|
| [v2-fabric-sp-procedures-name-arguments](v2-fabric-sp-procedures-name-arguments.md) | dbt-labs/dbt | Fabric passes names to `sp_tables`/`sp_columns` unquoted and as `LIKE` patterns | Measured on SQL Server, not Fabric |
| [v2-fabric-multi-statement-fetch-result-first-vs-last](v2-fabric-multi-statement-fetch-result-first-vs-last.md) | dbt-labs/dbt | Fabric is split and returns the last result; dbt-fabric 1.x (mssql-python) returned the first | Not run on Fabric |
| [v2-unit-test-fixture-cast-not-null](v2-unit-test-fixture-cast-not-null.md) | dbt-labs/dbt | Unit-test fixtures cast to `<type> NOT NULL` for a NOT NULL `given` column | Measured on SQL Server only; reproduce on Postgres or Fabric |
| [v2-unit-test-given-relation-schema-fetch-error](v2-unit-test-given-relation-schema-fetch-error.md) | dbt-labs/dbt | `given` schema fetch errors lose the driver message (`FsError::with_context` replaces it) | Underlying failure not reproduced |
| [v1-adapters-dispatch-snapshot-check-cols-columns](v1-adapters-dispatch-snapshot-check-cols-columns.md) | dbt-labs/dbt-adapters | Dispatch the `check_cols` column read under a new name; dispatching the existing one wakes a dormant `exasol__` macro | — |
| [v2-dbt-utils-expression-is-true-unnamed-column-tsql](v2-dbt-utils-expression-is-true-unnamed-column-tsql.md) | dbt-labs/dbt-utils | `expression_is_true` selects an unaliased literal; T-SQL rejects it | — |
| [v2-sqlserver-catalog-varchar-name-literals](v2-sqlserver-catalog-varchar-name-literals.md) | dbt-sqlserver-next/dbt-core | Catalog queries use `varchar` literals; non-code-page names are missed or mismatched | Fix on the branch |
| [v2-sqlserver-normalize-component-collation-fold](v2-sqlserver-normalize-component-collation-fold.md) | dbt-sqlserver-next/dbt-core | `normalize_component` lowercases regardless of collation | Needs a design decision |
| [v2-date-spine-nested-cte-tsql](v2-date-spine-nested-cte-tsql.md) | dbt-msft/dbt-sqlserver | `date_spine` fails on SQL Server (nested `WITH`, `order by 1`); `sqlserver__date_spine` override | — |
| [v1-varchar-name-literals-non-codepage](v1-varchar-name-literals-non-codepage.md) | dbt-msft/dbt-sqlserver | `varchar` name literals break table and incremental models named outside the code page | — |
| [v1-identifier-length-127-vs-128](v1-identifier-length-127-vs-128.md) | dbt-msft/dbt-sqlserver | `MAX_CHARACTERS_IN_IDENTIFIER` is 127; SQL Server allows 128; the columnstore index name can exceed 128 | — |
| [v1-use-database-deletes-embedded-quote](v1-use-database-deletes-embedded-quote.md) | dbt-msft/dbt-sqlserver | `get_use_database_sql` strips `"` instead of escaping it | — |
| [v1-use-database-state-vs-unqualified-catalog-reads](v1-use-database-state-vs-unqualified-catalog-reads.md) | dbt-msft/dbt-sqlserver | Two catalog reads ignore their database argument and follow the last `USE` | — |
| [v1-utils-macros-edge-cases](v1-utils-macros-edge-cases.md) | dbt-msft/dbt-sqlserver | `date_trunc` returns `DATE` below day; `split_part` fails on `&` and cuts parts at 128; `len` drops trailing spaces; `listagg` ignores `limit_num` | — |
| [v1-type-expansion-row-count](v1-type-expansion-row-count.md) | dbt-msft/dbt-sqlserver | The safe-expansion row count runs on every incremental and snapshot run | — |
| [v1-database-and-object-name-quoting](v1-database-and-object-name-quoting.md) | dbt-msft/dbt-sqlserver | Names with `;`, `'`, `-` or `&` break the connection string, `drop_schema`, `get_provision_sql`, `get_tables_by_pattern_sql` and the `drop_*` macros | — |
| [v1-mssql-python-minimum-version](v1-mssql-python-minimum-version.md) | dbt-msft/dbt-sqlserver | The `mssql` extra allows mssql-python versions (1.7.1 to 1.14.0) that fail seeds with 16-digit decimals | — |
| [v1-show-trailing-order-by](v1-show-trailing-order-by.md) | dbt-msft/dbt-sqlserver | `dbt show` fails when the query ends in `order by` | — |
| [v1-persist-docs-truncation](v1-persist-docs-truncation.md) | dbt-msft/dbt-sqlserver | `persist_docs` cuts descriptions at 3750 characters without a warning | — |
| [v1-readme-document-known-limits](v1-readme-document-known-limits.md) | dbt-msft/dbt-sqlserver | README states no limits for `hash`, `split_part` type, `listagg` size, snapshot hash and data-test views | — |

## Filed, open

| Draft | Live | What it is | Pending |
|---|---|---|---|
| [v2-ephemeral-select-wrapper-derived-table-alias](v2-ephemeral-select-wrapper-derived-table-alias.md) | [dbt-labs/dbt#16505](https://github.com/dbt-labs/dbt/issues/16505) | Ephemeral CTE injection wraps the model in an unaliased derived table; T-SQL rejects it (Msg 102) | Untriaged |
| [v2-unit-test-overrides-ignored](v2-unit-test-overrides-ignored.md) | [dbt-labs/dbt#16504](https://github.com/dbt-labs/dbt/issues/16504) | v2 never runs `unit` materialization or `get_unit_test_sql` overrides, from a project or an adapter; 1.x does | Untriaged |
| [dbt-core-sqlserver-v2-upstream-pr](dbt-core-sqlserver-v2-upstream-pr.md) | [dbt-labs/dbt#15769](https://github.com/dbt-labs/dbt/pull/15769) | The adapter PR (draft) | Live body synced on `4d70a1069` (#26 merged); needs a maintainer to label CI |
| [dbt-core-sqlserver-v2-bootstrap](dbt-core-sqlserver-v2-bootstrap.md) | [dbt-labs/dbt#15714](https://github.com/dbt-labs/dbt/issues/15714) | Scope issue the PR closes | Live body behind the draft (quoting, init SQL) |
| [dbt-sqlserver-v2-migration-tracking](dbt-sqlserver-v2-migration-tracking.md) | [dbt-msft/dbt-sqlserver#786](https://github.com/dbt-msft/dbt-sqlserver/issues/786) | Tracking index for the whole migration | Live body behind the draft |
| [v1-core-run-operation-never-commits](v1-core-run-operation-never-commits.md) | [dbt-labs/dbt#16434](https://github.com/dbt-labs/dbt/issues/16434) | `run-operation` never commits; `statement()` and `--sql` writes roll back on success | Untriaged |
| [v1-core-unit-test-cleanup-rolled-back](v1-core-unit-test-cleanup-rolled-back.md) | [dbt-labs/dbt#16499](https://github.com/dbt-labs/dbt/issues/16499) | `unit` materialization never commits; its temp-table drop rolls back and leaves `__dbt_tmp` behind | Untriaged |
| [v1-unit-test-temp-table-left-behind](v1-unit-test-temp-table-left-behind.md) | [dbt-msft/dbt-sqlserver#874](https://github.com/dbt-msft/dbt-sqlserver/issues/874) | With transactions on, every unit test leaves an empty `__dbt_tmp` table; adapter wraps the `unit` materialization and commits | Fix PR #875 open |
| [v1-column-ddl-on-schema-change](v1-column-ddl-on-schema-change.md) | [dbt-msft/dbt-sqlserver#881](https://github.com/dbt-msft/dbt-sqlserver/issues/881) | Widening a column drops `NOT NULL`; a snapshot can't add a column that needs quoting | Fix PR #884 open; 1.11 backport in #887 |
| [v1-delete-insert-predicates-alias](v1-delete-insert-predicates-alias.md) | [dbt-msft/dbt-sqlserver#882](https://github.com/dbt-msft/dbt-sqlserver/issues/882) | `delete+insert` predicates can't use `DBT_INTERNAL_DEST` | Fix PR #885 open; master only |
| [v1-seed-column-types](v1-seed-column-types.md) | [dbt-msft/dbt-sqlserver#883](https://github.com/dbt-msft/dbt-sqlserver/issues/883) | Seeds overflow `int` and `varchar(8000)` | Fix PR #886 open; 1.11 backport in #887 |

## Closed

| Draft | Live | Resolution |
|---|---|---|
| [v2-unit-test-actual-cte-nests-model-with-clause-tsql](v2-unit-test-actual-cte-nests-model-with-clause-tsql.md) | not filed | Fixed on the port instead: a `SqlServer` branch in `render_unit_test` lifts the model's CTEs into the unit test's `WITH` list, with no DDL ([fork #26](https://github.com/dbt-sqlserver-next/dbt-core/pull/26)). The general question, v2 ignoring `unit`/`get_unit_test_sql` overrides, is still to be measured and drafted |
| [v2-execute-inner-last-statement-clobbers-fetch-result](v2-execute-inner-last-statement-clobbers-fetch-result.md) | [dbt-labs/dbt#15765](https://github.com/dbt-labs/dbt/issues/15765) | Closed as not planned with PR #15766: SQL Server sends each batch whole (#23), and #15766's rule matches no 1.x driver |
| [v1-snapshot-check-cols-with-headed-query](v1-snapshot-check-cols-with-headed-query.md) | [dbt-msft/dbt-sqlserver#865](https://github.com/dbt-msft/dbt-sqlserver/issues/865) | Fixed in #867 |
| [v1-run-operation-writes-rolled-back](v1-run-operation-writes-rolled-back.md) | [dbt-msft/dbt-sqlserver#862](https://github.com/dbt-msft/dbt-sqlserver/issues/862) | Stopgap in #866 commits run-operation connections on success; remove once dbt-labs/dbt#16434 covers both paths |
| [v2-test-result-bool-parsing-truthiness](v2-test-result-bool-parsing-truthiness.md) | [dbt-labs/dbt#15767](https://github.com/dbt-labs/dbt/issues/15767) | Fixed upstream in `712702b7e`; PR #15768 closed unmerged |
| [v1-schema-concat-default-flip-for-v2-bridge](v1-schema-concat-default-flip-for-v2-bridge.md) | [dbt-msft/dbt-sqlserver#800](https://github.com/dbt-msft/dbt-sqlserver/issues/800) | Shipped in #811 (v1.12.0rc3) |
| [v1-quoting-alignment-with-v2](v1-quoting-alignment-with-v2.md) | [dbt-msft/dbt-sqlserver#785](https://github.com/dbt-msft/dbt-sqlserver/issues/785) | Shipped in #795 (v1.12.0rc2) |
| [dbt-core-sqlserver-v2-parts](dbt-core-sqlserver-v2-parts.md) | [dbt-sqlserver-next/dbt-core#1–#10](https://github.com/dbt-sqlserver-next/dbt-core/issues?q=is%3Aissue+Part) | All ten Part issues closed; the work is on `sqlserver-v2-port` |
