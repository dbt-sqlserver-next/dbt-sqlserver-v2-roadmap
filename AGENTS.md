# AGENTS.md

Conventions for agents and contributors in this roadmap repo. It plans the port
of dbt-sqlserver to dbt Core v2 (Fusion) and holds no adapter code; `plan/` and
`issues/` make claims about two codebases that keep moving:

```
dbt-labs/dbt:main ───── upstream remote ──┐
                                          ▼
dbt-sqlserver-next/dbt-core:sqlserver-v2-port  →  dbt-core/       PR #15769 → dbt-labs/dbt:main
dbt-msft/dbt-sqlserver:master (v1)             →  dbt-sqlserver/  the adapter being ported
```

`make setup` clones both (gitignored, not submodules; `plan/README.md` says
why). `.devcontainer/` carries both toolchains and runs `make setup` on
creation. Inside a checkout, its own `AGENTS.md` comes first; this file adds
only what the port needs.

The failure mode here is rarely a bug. It's a confident sentence that stopped
being true.

## Verify, don't recall

- Settle a claim with a command and state the result. No "very likely" or
  "should be" about something a `grep` answers.
- List a directory with `find <dir> -type f` before characterizing it. A
  partial grep once counted 2 files in `dbt-fabric`'s macro package; it holds 42.
- Cite the file and symbol, not the line: `relation_impl.rs` `include_policy`.
  Line numbers go stale silently, and a stale one still looks checked.
- Follow the call to its implementation. `adapter.parse_index` is dispatched from
  Jinja and resolves to `unimplemented!()`.
- Check live state before relying on or restating a decision. Read PR bodies,
  not just titles; #795's body held the live-server measurement that answered an
  open question.

```bash
gh issue view 786 --repo dbt-msft/dbt-sqlserver --json state,body
gh pr list --repo dbt-msft/dbt-sqlserver --state all --limit 20
gh issue list --repo dbt-sqlserver-next/dbt-core --state all
gh pr view 15769 --repo dbt-labs/dbt --json state,mergeable,comments
git -C dbt-core fetch upstream && git -C dbt-sqlserver fetch origin
```

When reality has moved, fix the affected docs in the same change and say so in
the commit message.

## Ask when a decision is ambiguous

Trade-offs between deferring work and shipping a silent behavior change belong
to the user. Ask before writing a plan section that assumes an answer, and give
enough to decide without opening a checkout:

1. What you found, with the file and symbol or PR.
2. What each option costs: crates, rough size, who blocks it.
3. The user-visible consequence, especially a silent one. A config that no-ops
   without an error is always worth flagging.
4. Your recommendation.

## Outward-facing actions

- Commit locally; don't push unless told to in the current session.
- Don't edit live issues or PRs on `dbt-msft/dbt-sqlserver`,
  `dbt-sqlserver-next/dbt-core` or `dbt-labs/dbt`. Update the draft in
  `issues/`, propose the edit, and note any drift between a draft and its live
  issue.

## Text

Brief by default: docs, drafts, commits, issues and PR descriptions. Cut any
sentence that repeats the one before it. When the point is a structure (branch
topology, a state machine, a build path), draw a diagram instead of describing
it, as above; when it isn't, leave the diagram out.

- **`plan/` and `issues/`** state what is true now. History lives in git, not in
  "previously said" or "updated on" notes. Don't hedge what you checked or
  assert what you didn't. Keep the audit hashes in `plan/00-current-state.md`
  honest: bump them only after re-verifying.
- **Commits** follow Conventional Commits: `type(scope): imperative subject`,
  lowercase, no period, under ~70 characters. The body carries only what the
  diff can't: why this shape, what was rejected, what's open.
- **PR bodies** group by decision, not by file, one line per non-obvious call
  with what was checked. State what you verified (`cargo test -p dbt-adapter
  --lib: 891 passed`), not what should work. No invented issue references, no
  `Signed-off-by` unless asked.

## Code

Write the simplest change that solves the problem. Avoid workarounds,
speculative abstractions and options no caller uses. In a checkout, match the
surrounding code and follow that repo's `AGENTS.md`.

## Comments

Only write what the code beside the comment cannot show. Don't narrate history
or restate the code. Do keep the gotchas: why a call runs once and not twice,
which direction a check fails, why a value is pinned or escaped. If a comment
asserts a deliberate choice, update it when the choice changes.

Code in `dbt-core/` and `dbt-sqlserver/` is read by people who don't know this
roadmap exists:

- Match the density of the neighbouring code. A five-line note beside
  one-liners reads as a warning sign.
- Keep the archaeology out: why v1 chose 127, what a constant was copied from,
  which alternative lost. That goes in the commit, the PR body and `plan/`.
- No comparisons with a neighbouring adapter unless the file already makes them;
  they go stale when the neighbour moves.
- Don't describe behavior that isn't implemented yet.

## Tests

- Verify SQL behavior by running it against a live server (`make server`).
  Assert on emitted SQL only for a property the server cannot show. Never assert
  on SQL instead of executing it. The same applies to claims in `plan/` and
  `issues/`: measure them, and name the server version.
- Test behavior and integration; the flow already exercises parameters and
  configs. Don't write trivial assertions (text in text, param in params),
  redundant checks, or tests that cannot fail.
- Keep throwaway checks and scratch databases out of the repo, and drop them
  when done.

## Where things live

- `plan/00`–`05`: audit, architecture, checklist, macro map, testing, decisions
  and risks. Start at `plan/README.md`.
- `issues/`: drafts of issues filed on other repos. Frontmatter carries
  `target_repo`, `status` (`draft`, `open` or `closed`) and the live `url`.
  `issues/README.md` indexes them by status; update it with any `status` change.
- `Makefile`: clones both repos and runs SQL Server through dbt-sqlserver's
  `docker-compose.yml`.
