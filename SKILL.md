---
name: py-init
description: Use when the user types `uv init`, `py init`, or otherwise starts a new Python project or fresh repo; asks to scaffold a CLI/library/FastAPI/ML project; or asks how to set up Python linting, formatting, type checking, testing, or pre-commit hooks from scratch — even if they don't name the tools.
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

   **Validate the name before running anything.** uv derives a module name
   from the distribution name but does **not** check the result is a legal
   Python identifier — reject up front, before `uv init`, any name whose
   module form starts with a digit, is a Python keyword, or doesn't match
   `[a-z_][a-z0-9_]*`. Ask for a different name; don't scaffold and hope.
   Applies to the *directory* name too in the current-directory case. See
   `references/pitfalls.md#naming-step-2` for how uv normalizes names and
   what breaks if you skip this.
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
`requires-python` and refuses anything outside it, and recovering from that
leaves the two files easy to mismatch. Passing `--python` up front avoids the
whole sequence — see `references/pitfalls.md#python-version-pinning-step-3`
for the failure mode this sidesteps.

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

**`known-first-party` takes the module name, not the project name** — get it
from the directory that exists, don't retype the project name (see
`references/pitfalls.md#known-first-party-vs-project-name-step-5` for what
silently breaks if you do):

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

**No upper bounds on the install command** — an upper-bounded install caps
resolution at today's versions, so a project scaffolded a year later gets
year-old tools. Install unbounded so today's project gets today's tools (see
`references/pitfalls.md#why-install-unbounded-then-pin-bounded-step-6`).

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

## Step 8 — Write the README

`uv init` leaves `README.md` as a bare `# <name>` heading. Replace it with:

```markdown
# <project name>

## Development setup

\`\`\`shell
uv sync
uv run pre-commit install --install-hooks \
  --hook-type pre-commit --hook-type pre-push --hook-type commit-msg
\`\`\`

## Verify setup

\`\`\`shell
uv run ruff check .
uv run ruff format --check .
uv run mypy
uv run pytest
\`\`\`
```

The project name heads the file — use the distribution name from `pyproject.toml`,
not the module name. The development-setup block mirrors Step 7. The verify
block is deliberately the non-mutating form (`ruff format --check`, no
`--fix`) — it's for a user confirming their clone already passes, not for
scaffolding-time fixing, which is what Step 10 does. Keep both blocks in sync
if the underlying steps' commands change.

## Step 9 — Check for placeholder leakage

Before running anything:

```shell
grep -rn --include='*.toml' --include='*.yaml' -e 'myapp' -e 'mylib' .
```

Any hit outside a comment means a step-5 substitution was missed. Fix it and
grep again.

This check exists because nothing downstream catches it. A stale
`known-first-party` or `coverage source` produces a perfectly green run while
sorting imports into the wrong group and measuring coverage on nothing.

## Step 10 — Verify

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

## Notes worth passing on, and what to add later

See `references/pitfalls.md` for CI/uv, the `ml` profile's conda boundary, uv's
pre-1.0 status, scaling the setup down for small projects, and optional
additions (`pydantic-settings`, `structlog`, `uv build`/`uv publish`).

## Daily commands (for the user, after setup)

```shell
uv add <pkg>                       # runtime dep
uv add --dev <pkg>                 # dev tool
uv lock --upgrade-package <pkg>    # upgrade one
uv sync                            # after a lockfile change
uv run pre-commit run              # before committing
```
