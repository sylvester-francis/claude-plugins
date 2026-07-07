---
name: refactoring-python
description: Use when refactoring or reviewing Python for clarity and type safety — adding type hints, dataclasses, comprehensions/generators, pathlib, context managers, fixing mutable defaults and bare excepts, and running ruff + mypy.
---

# Refactoring & reviewing Python

## Overview

Refactoring is **behavior-preserving** — keep tests green, change one thing at a time. The biggest wins in Python are **types the checker enforces** and letting **ruff** auto-fix the mechanical issues so review goes to design. Turn strictness up first and fix against the error list.

## Run the tools first

`ruff check --fix` + `ruff format` (replaces flake8/isort/black), then `mypy --strict` (or pyright). Anything they enforce shouldn't cost review time; `ruff` rules `B`, `SIM`, `UP`, `RUF` catch many anti-patterns automatically.

## High-value fixes

**Type the public surface, then let mypy find bugs.**
```python
def load(path):          ->   def load(path: Path) -> Config:
```
Strict mode surfaces the `None`/`Any` paths you'd otherwise hit at runtime.

**Dataclasses / NamedTuple over `__init__` boilerplate and dict-typed data.**
```python
@dataclass(frozen=True, slots=True)
class Point:
    x: int
    y: int
# vs a hand-written __init__, or passing {"x": 1, "y": 2} dicts around
```
`frozen=True` for value objects; `NamedTuple`/`TypedDict` when you need tuple/dict shape with types.

**Comprehensions / generators over C-style loops.** Build with a comprehension; stream large data with a generator (constant memory) instead of building a list.
```python
names = [u.name for u in users if u.active]        # not: append in a for-loop
total = sum(x.cost for x in items)                  # generator: no intermediate list
```

**`pathlib` over `os.path`; context managers for resources.**
```python
data = (Path(root) / name).read_text()             # not os.path.join + open/read/close
with open(p) as f: ...                              # not manual open/close
```

**Fix the mutable-default-arg trap.**
```python
def f(items=[]):          ->   def f(items: list[T] | None = None):
    ...                            items = items or []
```

**Narrow exceptions; EAFP.** No bare `except:` — catch the specific type; `raise New(...) from e`. Prefer try/except over pre-checking (LBYL) for the common Python idiom.

**Modern idioms.** f-strings (not `%`/`.format`); `enumerate`/`zip` (not index arithmetic); `dict`/`set` for membership (not list scans); `match` for tagged shapes; `|` unions and built-in generics (`list[int]`) on 3.10+.

## Reviewing a diff

Read for: an untyped public function; a `dict`/`tuple` passed where a dataclass belongs; a bare `except` swallowing errors; a mutable default; a list built where a generator would stream; `os.path` string-mangling; a resource opened without `with`. Confirm tests cover the change and `mypy --strict` + `ruff` pass.

## The discipline

Tests green before/after · one behavior-preserving change per commit · `ruff` + `mypy --strict` clean · prefer deleting code. A change that alters behavior is a feature/fix with its own test, not a refactor.

## Common mistakes

- "Refactoring" that quietly changes behavior with no test.
- Adding types as `Any`/`# type: ignore` instead of fixing the real type.
- `@dataclass` without `frozen`/`slots` for a value object; dict-typed data that wants a `TypedDict`/dataclass.
- Bare `except:` (hides `KeyboardInterrupt`/bugs); mutable default args.
- Building lists in a loop where a comprehension or generator is clearer and cheaper.
