# py-init

A [Claude Skill](https://code.claude.com/docs/en/skills) that scaffolds a Python
project with the modern toolchain — **uv, ruff, mypy, pytest, pre-commit** — all
configured in a single `pyproject.toml`.

Ask Claude *"set up a new Python project called foo"* and you get a repo that
formats, lints, type-checks and tests itself, with no back-and-forth about
which config keys go where.

Based on [How I Set Up a Python Project in 2026](https://akakritagya.hashnode.dev/how-i-set-up-a-python-project-in-2026-uv-ruff-mypy-and-friends).

---

## Install

### Claude Code — personal (all your projects)

```bash
git clone https://github.com/akakritagya/py-init ~/.claude/skills/py-init
```

### Claude Code — project (commit it, whole team gets it)

```bash
git clone https://github.com/akakritagya/py-init .claude/skills/py-init
```

Restart Claude Code, then check it loaded by asking what skills are available,
or invoke it directly with `/py-init`.

### claude.ai and other agents

This is a plain [Agent Skill](https://code.claude.com/docs/en/skills) —
`SKILL.md` plus a few asset files, nothing Claude-Code-specific. claude.ai
supports the same format (Settings → Capabilities → Skills), so uploading this
folder there works unmodified, as it runs in a sandboxed environment with
bash and file access.

For claude.ai (or anything that takes a packaged upload rather than a git
clone), grab the `py-init.skill` zip from the
[latest release](https://github.com/akakritagya/py-init/releases/latest)
instead — it's just `SKILL.md` and `assets/`, with the repo's README, LICENSE
and git metadata stripped out.

For any other AI coding agent: it'll work if that agent (a) reads `SKILL.md`
the way Claude Code and claude.ai do, and (b) gives the model bash and
filesystem tools to run `uv`, `ruff`, `mypy`, etc. If it doesn't support
Skills at all, this won't plug in as-is.

### Requirements

[uv](https://docs.astral.sh/uv/) on your PATH. Everything else the skill
installs itself.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh      # macOS / Linux
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"   # Windows
```

**On WSL**, install uv inside WSL — a Windows-side uv is invisible to your Linux
shell. Keep projects in the Linux filesystem (`~/projects`) rather than
`/mnt/c/`; uv creates a `.venv` with thousands of small files, and cross-
filesystem I/O will make it feel broken.

---

## What it does

The skill asks three questions — **project name**, **profile**, **Python
version** — then runs a fixed sequence:

1. Guards against scaffolding over an existing project
2. `uv python install` + `uv init --python` (app or lib layout)
3. Appends the profile's ruff / mypy / pytest config to `pyproject.toml`
4. Substitutes the per-project values that models usually get wrong
5. Installs tools **unbounded**, records **bounded** pins
6. Writes `.pre-commit-config.yaml`, runs `autoupdate`, installs all three hook stages
7. Greps for leftover placeholders
8. Verifies: `ruff check --fix` → `ruff format` → `mypy` → `pytest`

## Three profiles

| Profile | For | What changes |
|---|---|---|
| `strict` | libraries, long-lived code | everything on: `D` docstrings, `ANN`, `mypy strict`, warnings-as-errors |
| `app` | services, CLIs (**default**) | no global docstring rules |
| `ml` | experiments, training code | `mypy strict = false`, no `D`/`ANN`, warnings not promoted to errors, `data/` excluded |

The `ml` profile exists because `filterwarnings = ["error"]` turns every
numpy/torch/sklearn deprecation into a red suite on someone else's release
schedule, and strict typing on experiment code costs more than it catches.

## Design notes

Things that took testing to get right, in case you fork this:

- **Config lives in `assets/` as literal files**, not as prose in `SKILL.md`.
  A model that regenerates config from a description will eventually write
  `exclude_lines` instead of `exclude_also` — which *replaces* coverage's
  defaults rather than extending them.
- **Each profile has complete files**, not a base plus a diff. Duplication
  across three directories is cheaper than a model editing TOML at scaffold time.
- **Install unbounded, record bounded.** `uv add --dev ruff` with no ceiling, so
  a project scaffolded next year gets next year's tools; then pin
  `>=<resolved>,<<next major>` so CI stays reproducible.
- **`--python` goes on `uv init`**, not a separate `uv python pin` afterwards.
  Pinning after init hits `error: The requested Python version is incompatible
  with the project requires-python value` and the recovery leaves
  `.python-version` and `requires-python` disagreeing.
- **`known-first-party` takes the module name**, read off `src/` — not what the
  user typed. For `name-test` the module is `name_test`, and the hyphenated form
  means isort silently sorts your own imports into the third-party block.

## Customising

Most changes belong in `assets/`, not `SKILL.md`. Line length, selected rule
sets, coverage excludes — edit the profile file and it takes effect on the next
run.

The `ml` profile's `ignore_missing_imports` list (sklearn, scipy, matplotlib,
datasets, transformers, seaborn, statsmodels, networkx, xgboost, lightgbm,
cv2, nltk, tensorflow, keras) is a starting guess. Swap in whatever you
actually import, or it's dead config that could mask a real typo.

## Licence

MIT — see [LICENSE](LICENSE).
