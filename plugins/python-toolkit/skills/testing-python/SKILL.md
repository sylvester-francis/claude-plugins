---
name: testing-python
description: Use when writing or improving Python tests — pytest structure, fixtures and conftest, parametrized table tests, fakes vs monkeypatch vs mock, testing HTTP, tmp_path, coverage, and property-based tests with Hypothesis.
---

# Testing Python

## Overview

Test the **installed package** (that's why `src/` layout matters) with **pytest**. Default to **fakes over mocks**, **parametrized** cases, and dependencies behind a `Protocol` so faking is trivial. Assert on outcomes and state so tests survive refactors.

## Layout & fixtures

`tests/` next to `src/`; shared fixtures in `conftest.py` (auto-discovered, no import). Fixtures build dependencies; scope them (`function` default, `session` for expensive read-only). Use `tmp_path`/`tmp_path_factory` for filesystem work — never write to the repo.

## Dependencies: fakes over mocks

Define the dependency as a `Protocol` the consumer needs; hand-write an in-memory fake and assert on state. Reach for `unittest.mock`/`pytest-mock`'s `mocker` only when the *interaction* is the behavior ("charged once") or to inject a failure; `monkeypatch` for env vars, attributes, and `setattr` on a module.

```python
from typing import Protocol

class Store(Protocol):
    def get(self, id: str) -> User | None: ...
    def put(self, u: User) -> None: ...

class FakeStore:                     # a real, minimal implementation
    def __init__(self) -> None: self._d: dict[str, User] = {}
    def get(self, id: str) -> User | None: return self._d.get(id)
    def put(self, u: User) -> None: self._d[u.id] = u
```

## Parametrized table tests

```python
import pytest

@pytest.mark.parametrize("email, ok", [
    ("a@b.co", True), ("bad", False), ("", False),
])
def test_valid_email(email: str, ok: bool) -> None:
    assert is_valid(email) is ok
```
`ids=` for readable names; `pytest.param(..., marks=pytest.mark.xfail)` for known-broken rows. Compare exceptions with `pytest.raises(ValueError, match="...")`.

## HTTP & integration

Fake the client behind a `Protocol`, or use `responses`/`respx` to stub `requests`/`httpx` at the transport. Keep real-DB/real-network tests in a separate suite gated by a marker (`@pytest.mark.integration`) or `Testcontainers`, excluded from the fast default run. The service test wires the **real service + fake store**; the integration test exercises the **real adapter**.

## Property-based (Hypothesis)

For invariants ("round-trips", "never crashes", "always sorted"), let Hypothesis generate inputs:
```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs: list[int]) -> None:
    assert my_sort(my_sort(xs)) == my_sort(xs)
```

## Running

```bash
pytest -q                         # quiet
pytest -x -k "email and not slow" # stop on first fail, filter by name
pytest --cov=myproj --cov-report=term-missing --cov-fail-under=85
```
Wire `pytest --cov` into CI. `-p no:randomly`/`pytest-randomly` surfaces order coupling.

## Common mistakes

- Flat layout → tests import the working tree, not the installed wheel (`src/` fixes it).
- Mocking call sequences instead of asserting outcomes → brittle tests.
- Sharing mutable fixtures across tests → build fresh in the fixture.
- Real I/O in unit tests → fake it; mark real-dependency tests `integration` and exclude by default.
- `assert x == True`/broad `except` in tests → assert the value; use `pytest.raises`.
