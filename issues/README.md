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

## Not filed yet

| Draft | Target | What it is | Before filing |
|---|---|---|---|
| [v2-fabric-sp-procedures-name-arguments](v2-fabric-sp-procedures-name-arguments.md) | dbt-labs/dbt | Fabric passes names to `sp_tables`/`sp_columns` unquoted and as `LIKE` patterns | Measured on SQL Server, not Fabric |
| [v2-unit-test-actual-cte-nests-model-with-clause-tsql](v2-unit-test-actual-cte-nests-model-with-clause-tsql.md) | dbt-labs/dbt | Unit-test SQL nests a model's `WITH` inside a CTE; T-SQL rejects it | Hoist fix measured as SQL, not implemented |
| [v2-unit-test-given-relation-schema-fetch-error](v2-unit-test-given-relation-schema-fetch-error.md) | dbt-labs/dbt | `given` schema fetch errors lose the driver message (`FsError::with_context` replaces it) | Underlying failure not reproduced |
| [v2-dbt-utils-expression-is-true-unnamed-column-tsql](v2-dbt-utils-expression-is-true-unnamed-column-tsql.md) | dbt-labs/dbt-utils | `expression_is_true` selects an unaliased literal; T-SQL rejects it | — |
| [v2-sqlserver-catalog-varchar-name-literals](v2-sqlserver-catalog-varchar-name-literals.md) | dbt-sqlserver-next/dbt-core | Catalog queries use `varchar` literals; non-code-page names are missed or mismatched | Fix on the branch |
| [v2-sqlserver-normalize-component-collation-fold](v2-sqlserver-normalize-component-collation-fold.md) | dbt-sqlserver-next/dbt-core | `normalize_component` lowercases regardless of collation | Needs a design decision |
| [v2-date-spine-nested-cte-tsql](v2-date-spine-nested-cte-tsql.md) | dbt-msft/dbt-sqlserver | `date_spine` fails on SQL Server (nested `WITH`, `order by 1`); `sqlserver__date_spine` override | — |
| [v1-varchar-name-literals-non-codepage](v1-varchar-name-literals-non-codepage.md) | dbt-msft/dbt-sqlserver | `varchar` name literals break table and incremental models named outside the code page | — |
| [v1-identifier-length-127-vs-128](v1-identifier-length-127-vs-128.md) | dbt-msft/dbt-sqlserver | `MAX_CHARACTERS_IN_IDENTIFIER` is 127; SQL Server allows 128 | — |
| [v1-use-database-deletes-embedded-quote](v1-use-database-deletes-embedded-quote.md) | dbt-msft/dbt-sqlserver | `get_use_database_sql` strips `"` instead of escaping it | — |
| [v1-use-database-state-vs-unqualified-catalog-reads](v1-use-database-state-vs-unqualified-catalog-reads.md) | dbt-msft/dbt-sqlserver | Two catalog reads ignore their database argument and follow the last `USE` | — |

## Filed, open

| Draft | Live | What it is | Pending |
|---|---|---|---|
| [dbt-core-sqlserver-v2-upstream-pr](dbt-core-sqlserver-v2-upstream-pr.md) | [dbt-labs/dbt#15769](https://github.com/dbt-labs/dbt/pull/15769) | The adapter PR (draft) | Rebased on `main` and pushed (`46704326d`); live body behind the draft |
| [dbt-core-sqlserver-v2-bootstrap](dbt-core-sqlserver-v2-bootstrap.md) | [dbt-labs/dbt#15714](https://github.com/dbt-labs/dbt/issues/15714) | Scope issue the PR closes | Live body behind the draft (quoting, init SQL) |
| [v2-execute-inner-last-statement-clobbers-fetch-result](v2-execute-inner-last-statement-clobbers-fetch-result.md) | [dbt-labs/dbt#15765](https://github.com/dbt-labs/dbt/issues/15765) | Multi-statement batch returns the cleanup query's empty result | Fix PR #15766 unreviewed and conflicting; live body behind the draft |
| [dbt-sqlserver-v2-migration-tracking](dbt-sqlserver-v2-migration-tracking.md) | [dbt-msft/dbt-sqlserver#786](https://github.com/dbt-msft/dbt-sqlserver/issues/786) | Tracking index for the whole migration | Live body behind the draft |
| [v1-core-run-operation-never-commits](v1-core-run-operation-never-commits.md) | [dbt-labs/dbt#16434](https://github.com/dbt-labs/dbt/issues/16434) | `run-operation` never commits; a macro's `statement()` writes roll back on success | Untriaged |
| [v1-run-operation-writes-rolled-back](v1-run-operation-writes-rolled-back.md) | [dbt-msft/dbt-sqlserver#862](https://github.com/dbt-msft/dbt-sqlserver/issues/862) | With transactions on (1.12 default), `run-operation` writes are silently lost; workaround `adapter.commit_if_open()` | Waits on #16434 |

## Closed

| Draft | Live | Resolution |
|---|---|---|
| [v2-test-result-bool-parsing-truthiness](v2-test-result-bool-parsing-truthiness.md) | [dbt-labs/dbt#15767](https://github.com/dbt-labs/dbt/issues/15767) | Fixed upstream in `712702b7e`; PR #15768 closed unmerged |
| [v1-schema-concat-default-flip-for-v2-bridge](v1-schema-concat-default-flip-for-v2-bridge.md) | [dbt-msft/dbt-sqlserver#800](https://github.com/dbt-msft/dbt-sqlserver/issues/800) | Shipped in #811 (v1.12.0rc3) |
| [v1-quoting-alignment-with-v2](v1-quoting-alignment-with-v2.md) | [dbt-msft/dbt-sqlserver#785](https://github.com/dbt-msft/dbt-sqlserver/issues/785) | Shipped in #795 (v1.12.0rc2) |
| [dbt-core-sqlserver-v2-parts](dbt-core-sqlserver-v2-parts.md) | [dbt-sqlserver-next/dbt-core#1–#10](https://github.com/dbt-sqlserver-next/dbt-core/issues?q=is%3Aissue+Part) | All ten Part issues closed; the work is on `sqlserver-v2-port` |
