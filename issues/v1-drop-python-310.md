---
target_repo: dbt-msft/dbt-sqlserver
type: bug
status: open
url: https://github.com/dbt-msft/dbt-sqlserver/issues/897
---

# Remove Python 3.10 soft-installability now that it is end-of-life

Python 3.10 reached end of life on 2026-10-01 and no longer receives security fixes ([Python Developer's Guide](https://devguide.python.org/versions/)). The adapter's CI already tests Python 3.11 and newer, but package metadata and the README still permit or advertise Python 3.10. Issue [#758](https://github.com/dbt-msft/dbt-sqlserver/issues/758) was closed after PR [#762](https://github.com/dbt-msft/dbt-sqlserver/pull/762) dropped Python 3.10; PR [#778](https://github.com/dbt-msft/dbt-sqlserver/pull/778) later restored soft-installability until its scheduled EOL. That date has now passed.

## Steps to reproduce

- `pyproject.toml` sets `requires-python = ">=3.10"` and includes the Python 3.10 classifier.
- `README.md` lists Python 3.10 as installable.
- `uv.lock` records the same Python floor.

## Environment

- Adapter: dbt-sqlserver (v1)
- Python versions covered by CI: 3.11–3.14

Raise the minimum supported Python version to 3.11, remove the 3.10 classifier and README entry, and regenerate `uv.lock`. Check packaging and development metadata for any other active Python 3.10 support declarations; keep historical changelog entries intact. Verify the package metadata rejects Python 3.10 and accepts the supported versions.
