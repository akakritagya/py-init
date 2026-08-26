---
name: py-init
description: Scaffold a new Python project with the modern uv/ruff/mypy/pytest/pre-commit toolchain, fully configured in pyproject.toml. Use this whenever the user starts a new Python project, sets up a fresh repo, mentions `uv init`, asks to scaffold a CLI/library/FastAPI/ML project, or asks how to set up Python tooling — even if they don't name the tools. Also use when a Python project needs linting, formatting, type checking, testing, or pre-commit hooks added from scratch.
---

# py-init

Scaffold a Python project that formats, lints, types and tests itself.

Five tools, all configured in `pyproject.toml`:

| Tool | Job |
|---|---|
| uv | Python versions, environments, dependencies, lockfile |
| ruff | linter *and* formatter (replaces black, flake8, isort, pyupgrade, pydocstyle) |
| mypy | checks that annotations aren't lying |
| pytest | test runner + coverage |
| pre-commit | wires the other four to git |

Config text lives in `assets/`. **Copy those files; do not retype or
regenerate their contents.** Rewriting config from memory is how
`exclude_also` silently becomes `exclude_lines` (which replaces coverage's
defaults instead of extending them) and how `[[tool.mypy.overrides]]` loses a
bracket. The only edits to make are the substitutions listed in step 5 and 6.

## Step 1 — Guard

Check the target directory for `pyproject.toml` or `.git/`. If either exists,
**stop and ask the user** before touching anything. This skill scaffolds new
projects; it must never overwrite an existing one.

One legitimate exception: scaffolding into a freshly cloned empty repo, where
`.git/` exists by design and there's no `pyproject.toml`. Confirm that's the
situation and proceed. `pyproject.toml` present is never waved through.

## Step 2 — Ask three questions

1. **Project name** — or **current directory**. Two shapes:

   - **New subdirectory**: `uv init <name>` creates `<name>/` and scaffolds
     inside it.
   - **Current directory**: `uv init` with no name scaffolds in place and takes
     the project name from the directory. Right when the user has already made
     the folder, or cloned an empty repo and wants the project inside it.
     `--name <name>` overrides the derived name if the directory is called
     something else.

   **Validate the name before running anything.** uv normalizes the
   distribution name (lowercase, separators to hyphens) and derives the module
   name by swapping those back to underscores — but it does **not** check the
   result is a legal Python identifier:

   | Given | Distribution | Module dir |
   |---|---|---|
   | `svc` | `svc` | `src/svc/` |
   | `name-test` | `name-test` | `src/name_test/` |
   | `My_App` | `my-app` | `src/my_app/` |
   | `my.app` | `my-app` | `src/my_app/` |

   `uv init --app --package 2fast` and `... class` both scaffold cleanly and
   produce packages that raise `SyntaxError` on import — uv even writes
   `2fast = "2fast:main"` as the entry point. Nothing fails until someone
   imports it. So reject up front, before `uv init`, any name whose module form
   starts with a digit, is a Python keyword, or doesn't match
   `[a-z_][a-z0-9_]*`. Ask for a different name; don't scaffold and hope.

   In the current-directory case this applies to the *directory* name, which
   the user may not have chosen with Python in mind — `uv init` in a folder
   called `weird dir` yields distribution `weird-dir` and module `weird_dir`.
   Check it and offer `--name` if it doesn't work.
2. **Profile** — `strict`, `app`, or `ml`. See the table below. Default to
   `app` if the user doesn't care.
3. **Python version.** Run `uv python list` first so the user picks from what's
   actually available and downloadable rather than guessing:

   ```shell
   uv python list        # installed and downloadable builds
   ```

   Offer the newest stable build as the default. This needs no project to
   exist, so ask it here rather than after `uv init`.

Don't interview further. Everything else has a defensible default.

| Profile | For | What changes |
|---|---|---|
| `strict` | libraries, long-lived code | everything on: `D` docstring rules, `ANN`, `mypy strict`, warnings-as-errors |
| `app` | services, CLIs (**default**) | no global `D` rules; everything else as strict |
| `ml` | experiments, training code | `mypy strict = false`, no `D`/`ANN`, warnings not promoted to errors, `data/` and checkpoints excluded |

The `ml` profile exists because `filterwarnings = ["error"]` turns every
numpy/torch/sklearn deprecation into a red suite on somebody else's release
schedule, and strict typing on experiment code costs more than it catches.

## Step 3 — Install Python, then initialize

Install the chosen version **before** `uv init`, then hand it to `uv init`
directly:

```shell
uv python install <version>       # no-op if already installed

uv init --python <version> --app --package <name>   # application
uv init --python <version> --lib <name>             # library
```

Drop the trailing `<name>` to scaffold in the current directory instead, adding
`--name <name>` if the project should be called something other than the
folder. Everything downstream is identical either way.

Ask which layout if it isn't obvious from context. `--app --package` is the
common case; a project meant to be imported by others is `--lib`.

`--python` makes `uv init` write a consistent set in one shot:
`requires-python` in `pyproject.toml` and `.python-version` both land on the
chosen version, with nothing to reconcile afterwards.

**Don't init first and pin afterwards.** `uv python pin` validates against
`requires-python` and refuses anything outside it:

```
error: The requested Python version `3.13` is incompatible with the
project `requires-python` value of `>=3.14`.
```

Recovering from that means editing `requires-python` by hand before pinning,
and a half-done recovery leaves `.python-version` and `requires-python`
disagreeing — a mismatch mypy's `python_version` then inherits in step 5.
Passing `--python` up front avoids the whole sequence.

Confirm both files agree before moving on. Treat `requires-python`,
`.python-version`, and mypy's `python_version` as three names for one decision.

## Step 4 — Append config

Append to `pyproject.toml`, in this order:

1. `assets/profiles/<profile>/ruff.toml`
2. `assets/profiles/<profile>/mypy.toml`
3. `assets/pyproject-pytest.toml` — or `assets/pyproject-pytest-ml.toml` for
   the `ml` profile

Strip the leading comment block from each file as you append; the rest goes in
verbatim.

## Step 5 — Substitute per-project values

These are the spots that go wrong silently. Handle each one explicitly:

| Key | Set to |
|---|---|
| `[tool.ruff.lint.isort] known-first-party` | the **module** name — read it off `src/`, don't retype what the user said |
| `[tool.mypy] python_version` | the version chosen in step 2 (re-read `.python-version` to confirm) |
| `[project.scripts]` | already written by `uv init` for apps; confirm rather than rewrite |
| `[tool.coverage.run] source` | `["src"]` is correct for the default uv layout — change it only if the layout differs |

**`known-first-party` takes the module name, not the project name.** For a
project called `name-test`, the module is `name_test` — putting the hyphenated
form here means isort never matches it and sorts your own imports into the
third-party block. Silent, and a green run. Get it from the directory that
exists:

```shell
ls src/        # this is the name that goes in known-first-party
```

**`python_version` must match `.python-version`.** mypy does not read
`requires-python`, so a mismatch means you're type-checking against a different
language version than you run — and it fails in confusing ways, not obvious
ones.

`[tool.ruff] required-version` is also a substitution, but it depends on what
actually installs. It's handled in step 6.

## Step 6 — Install unbounded, record bounded

```shell
uv add --dev ruff mypy pytest pytest-cov pre-commit
```

**No upper bounds on the install command.** The article this skill is based on
shows `uv add --dev "ruff>=0.16.4,<0.17"` — that form caps resolution at
August-2026 versions, so a project scaffolded a year later gets year-old tools.
Install unbounded so today's project gets today's tools.

Then read what uv actually resolved (`uv pip list` or the `[dependency-groups]`
block) and record bounded pins:

- Pin each tool `>=<resolved>,<<next major>` in `pyproject.toml`. Unbounded is
  for *resolving*; bounded is for what gets written down, so CI and
  collaborators land on the same tool months from now.
- **Set `[tool.ruff] required-version` from the resolved ruff version.** The
  asset files carry the article's literal `">=0.16,<0.17"`. If ruff resolved
  outside that range, ruff refuses to run at all against the config — a hard,
  immediate failure, which is the good kind, but only if you fix it here.
- Same for the `uv_build` bound in `[build-system]`.

If a resolved version is **older** than the asset's value, say so rather than
silently downgrading the pin.

If a resolved version **crosses a major boundary** — ruff 0.17+, mypy 3.x,
pytest 10.x — widening the pin is not enough. Config keys move across those
boundaries. Tell the user explicitly which asset may need review; do not assume
the old block is still valid under the new major.

## Step 7 — pre-commit

Write `assets/pre-commit-config.yaml` to `.pre-commit-config.yaml` next to
`pyproject.toml`. Then:

```shell
uv run pre-commit autoupdate      # refresh rev: pins, don't trust the asset
uv run pre-commit install --install-hooks \
  --hook-type pre-commit --hook-type pre-push --hook-type commit-msg
```

All three hook types matter: `pre-commit` for ruff and mypy, `commit-msg` for
conventional commits, `pre-push` for the test suite. pytest is deliberately on
pre-push rather than pre-commit — a suite that runs on every commit is a suite
that gets `--no-verify`'d once it grows past a few seconds.

Note `pass_filenames: false` on the mypy hook: mypy needs the whole package to
resolve types, not just the staged files.

## Step 8 — Check for placeholder leakage

Before running anything:

```shell
grep -rn --include='*.toml' --include='*.yaml' -e 'myapp' -e 'mylib' .
```

Any hit outside a comment means a step-5 substitution was missed. Fix it and
grep again.

This check exists because nothing downstream catches it. A stale
`known-first-party` or `coverage source` produces a perfectly green run while
sorting imports into the wrong group and measuring coverage on nothing.

## Step 9 — Verify

Run in this exact order and report the results:

```shell
uv run ruff check . --fix
uv run ruff format .
uv run mypy
uv run pytest
```

**Order matters.** `check --fix` can leave lines that need re-wrapping, so
`format` goes after it. Running format first means the next check undoes your
formatting.

**Expect the `strict` profile to fail on uv's own boilerplate.** `uv init`
generates `src/<name>/__init__.py` with no module or function docstring, which
`D103`/`D104` flag immediately. Add real docstrings to the generated file
before declaring the run green — don't silence the rules to make the scaffold
pass. A first run that fails on generated code and then goes green after a
two-line fix is working correctly.

Also add a minimal `tests/` file so pytest has something to collect; an empty
run with `--strict-config` reports no tests rather than success.

## Notes worth passing on

- **CI needs uv.** The local pre-commit hooks use `language: system` with
  `uv run`, so any CI job running `pre-commit run --all-files` must install uv
  first.
- **uv doesn't replace conda** for CUDA toolkits and other non-Python binaries.
  Relevant to the `ml` profile: uv manages the Python side, the system or conda
  still provides the native stack.
- **uv is pre-1.0** (0.12.x at time of writing). Stable and widely used in
  production; the version number is misleading.
- **Not everything has to land on day one.** If the user wants less, start with
  `uv init` plus ruff and add the rest when it's needed. A setup nobody runs is
  worse than a smaller one they do.

## Worth adding later, not by default

- `pydantic-settings` — scattered `os.environ` calls become one typed object
  that fails at startup rather than at 3am.
- `structlog` — structured JSON logs an aggregator can query.
- `uv build` / `uv publish` with trusted publishing — no API token in repo
  secrets; a tag-triggered CI job turns releases into `git tag && git push --tags`.

## Daily commands (for the user, after setup)

```shell
uv add <pkg>                       # runtime dep
uv add --dev <pkg>                 # dev tool
uv lock --upgrade-package <pkg>    # upgrade one
uv sync                            # after a lockfile change
uv run pre-commit run              # before committing
```
