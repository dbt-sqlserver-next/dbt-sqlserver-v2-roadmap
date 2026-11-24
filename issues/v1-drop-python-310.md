---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: closed
url: https://github.com/dbt-msft/dbt-sqlserver/issues/897
pr: https://github.com/dbt-msft/dbt-sqlserver/pull/905
---

# Remove Python 3.10 soft-installability now that it is end-of-life

> **In process 2026-11-24** — [PR #905](https://github.com/dbt-msft/dbt-sqlserver/pull/905)
> (`Closes #897`) raises `requires-python` to `>=3.11` and drops the classifier
> and README row. Tracked here as closed; the live issue stays open until it
> merges.

Python 3.10 reached end of life on 2026-10-01 and no longer receives security fixes ([Python Developer's Guide](https://devguide.python.org/versions/)). The adapter's CI already tests Python 3.11 and newer, but package metadata and the README still permit or advertise Python 3.10. Issue [#758](https://github.com/dbt-msft/dbt-sqlserver/issues/758) was closed after PR [#762](https://github.com/dbt-msft/dbt-sqlserver/pull/762) dropped Python 3.10; PR [#778](https://github.com/dbt-msft/dbt-sqlserver/pull/778) later restored soft-installability until its scheduled EOL. That date has now passed.

## Steps to reproduce

- `pyproject.toml` sets `requires-python = ">=3.10"` and includes the Python 3.10 classifier.
- `README.md` lists Python 3.10 as installable.
- `uv.lock` records the same Python floor.

## Environment

- Adapter: dbt-sqlserver (v1)
- Python versions covered by CI: 3.11–3.14

Raise the minimum supported Python version to 3.11, remove the 3.10 classifier and README entry, and regenerate `uv.lock`. Check packaging and development metadata for any other active Python 3.10 support declarations; keep historical changelog entries intact. Verify the package metadata rejects Python 3.10 and accepts the supported versions.
