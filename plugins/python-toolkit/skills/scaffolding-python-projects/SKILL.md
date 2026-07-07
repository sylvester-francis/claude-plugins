---
name: scaffolding-python-projects
description: Use when creating, scaffolding, or initializing a new Python project — a CLI, service, or PyPI library — deciding the src/ layout, pyproject.toml, dependency manager (uv), linting/formatting (ruff), type checking (mypy), testing (pytest), and packaging.
---

# Scaffolding Python projects

## Overview

Stand up a Python project that is **typed, linted, and `src`-layout** from day one. Two decisions shape it: **application vs. library** (does anyone `pip install` and import it?), and **how strict** the type checker is (set it high now — retrofitting types is miserable).

**Core rule: `src/` layout, `pyproject.toml` as the single source of truth, ruff + mypy strict, pinned toolchain.**

## First decision: application or library?

| | Application (CLI / service) | Library (published to PyPI) |
|---|---|---|
| Consumed by | run as a program | `import`ed by other projects |
| Entry point | `[project.scripts]` → `app.__main__:main` | public API in `__init__.py` |
| Versioning | any | SemVer; `py.typed` marker so consumers get your types |
| Deps | pinned via lockfile | minimal, lower-bounded ranges in `[project.dependencies]` |

## Layout (both)

```
myproj/
  pyproject.toml           # PEP 621 metadata + tool config, the single source of truth
  uv.lock                  # committed lockfile (apps); libraries commit it too for dev
  .python-version          # pins the interpreter for uv/pyenv
  src/myproj/
    __init__.py            # library: the public API. app: package marker
    __main__.py            # app: `python -m myproj` entry
    py.typed               # library: ship types (PEP 561)
  tests/
    conftest.py
    test_*.py
  README.md  LICENSE
```

**`src/` layout is non-negotiable**: it stops tests from importing the un-installed working copy and forces you to test the installed package (catches missing-file-in-wheel bugs).

## Toolchain (recommended defaults)

- **uv** — the dependency manager + venv + runner. Fast, single tool: `uv init`, `uv add`, `uv run pytest`, `uv lock`, `uv sync`. Commit `uv.lock`. (Poetry/PDM/hatch are fine; uv is the fast modern default.)
- **ruff** — lint *and* format (replaces flake8 + isort + black). Enable a broad ruleset (`E,F,I,UP,B,SIM,RUF`) in `[tool.ruff.lint]`.
- **mypy** (or pyright) in **strict mode** — `[tool.mypy] strict = true`. This is where the compounding value is.
- **pytest** — `[tool.pytest.ini_options]`, tests in `tests/`, run with coverage.
- **Pin the interpreter** (`.python-version` / `requires-python`) and pin the toolchain versions.

### `pyproject.toml` skeleton
```toml
[project]
name = "myproj"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []
[project.scripts]           # apps only
myproj = "myproj.__main__:main"
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]
[tool.mypy]
strict = true
```

## Scaffold checklist

1. Decide **app vs. library**.
2. `uv init --package myproj` (or set up `pyproject.toml` by hand); `requires-python`.
3. `src/myproj/` + `tests/`; library adds `py.typed` and a real `__init__.py` API.
4. Configure ruff + mypy(strict) + pytest in `pyproject.toml`.
5. `.gitignore` (`.venv/`, `__pycache__/`, `dist/`, `*.egg-info/`, `.mypy_cache/`, `.pytest_cache/`); `README`, `LICENSE`.
6. CI: `uv sync` → `ruff check` + `ruff format --check` → `mypy` → `pytest --cov`; libraries also `uv build` + `twine check`.
7. First test, then commit.

## Common mistakes

- **Flat layout** (package at repo root) → tests import the source tree, not the installed wheel; use `src/`.
- **No `py.typed`** on a typed library → consumers get `Any` for your package.
- **`mypy` non-strict** → the null/`Any` bugs strict mode catches slip through.
- **`requirements.txt` + setup.py** for new work → use `pyproject.toml` + a lockfile.
- **Mutable default args**, `print`-debugging, and untyped public functions shipped as v1.
- **Not pinning the interpreter** → "works on my 3.13" drift.
