# py-init: pitfalls and rationale

Background for the rules in `SKILL.md`. Read a section when the linked step
sends you here — this file isn't meant to be read top to bottom.

## Naming (Step 2)

`uv init --app --package 2fast` and `... class` both scaffold cleanly and
produce packages that raise `SyntaxError` on import — uv even writes
`2fast = "2fast:main"` as the entry point. Nothing fails until someone
imports it.

uv normalizes the distribution name (lowercase, separators to hyphens) and
derives the module name by swapping those back to underscores — but it does
**not** check the result is a legal Python identifier:

| Given | Distribution | Module dir |
| --- | --- | --- |
| `svc` | `svc` | `src/svc/` |
| `name-test` | `name-test` | `src/name_test/` |
| `My_App` | `my-app` | `src/my_app/` |
| `my.app` | `my-app` | `src/my_app/` |

In the current-directory case the same check applies to the *directory*
name, which the user may not have chosen with Python in mind — `uv init` in
a folder called `weird dir` yields distribution `weird-dir` and module
`weird_dir`.

## Python version pinning (Step 3)

`uv python pin` validates against `requires-python` and refuses anything
outside it:

```text
error: The requested Python version `3.13` is incompatible with the
project `requires-python` value of `>=3.14`.
```

Recovering from that means editing `requires-python` by hand before
pinning, and a half-done recovery leaves `.python-version` and
`requires-python` disagreeing — a mismatch mypy's `python_version` then
inherits in step 5. Passing `--python` to `uv init` up front avoids the
whole sequence: it writes `requires-python` and `.python-version` from the
same value in one shot, with nothing to reconcile afterwards.

## known-first-party vs. project name (Step 5)

`known-first-party` takes the **module** name, not the project name. For a
project called `name-test`, the module is `name_test` — putting the
hyphenated form here means isort never matches it and sorts your own
imports into the third-party block. Silent, and a green run.

## Why install unbounded, then pin bounded (Step 6)

The article this skill is based on shows `uv add --dev "ruff>=0.16.4,<0.17"`
— that form caps resolution at August-2026 versions, so a project scaffolded
a year later gets year-old tools. Installing unbounded gets today's project
today's tools; the bounded pin is recorded *after* resolution so CI and
collaborators land on the same tool months from now.

## Notes worth passing on

- **CI needs uv.** The local pre-commit hooks use `language: system` with
  `uv run`, so any CI job running `pre-commit run --all-files` must install
  uv first.
- **uv doesn't replace conda** for CUDA toolkits and other non-Python
  binaries. Relevant to the `ml` profile: uv manages the Python side, the
  system or conda still provides the native stack.
- **uv is pre-1.0** (0.12.x at time of writing). Stable and widely used in
  production; the version number is misleading.
- **Not everything has to land on day one.** If the user wants less, start
  with `uv init` plus ruff and add the rest when it's needed. A setup nobody
  runs is worse than a smaller one they do.

## Worth adding later, not by default

- `pydantic-settings` — scattered `os.environ` calls become one typed object
  that fails at startup rather than at 3am.
- `structlog` — structured JSON logs an aggregator can query.
- `uv build` / `uv publish` with trusted publishing — no API token in repo
  secrets; a tag-triggered CI job turns releases into
  `git tag && git push --tags`.
