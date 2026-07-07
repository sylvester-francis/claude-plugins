# python-toolkit

A bundle of Python developer-productivity skills for Claude Code.

## Install

```
/plugin marketplace add sylvester-francis/claude-plugins
/plugin install python-toolkit@sylvester-plugins
```

## Skills

- **scaffolding-python-projects** — `src/` layout, `pyproject.toml`, uv, ruff,
  mypy (strict), pytest, and app-vs-PyPI-library packaging.
- **debugging-python** — tracebacks and exception chains, `pdb`/post-mortem,
  `py-spy`, `faulthandler`, `tracemalloc`, and asyncio task inspection.
- **testing-python** — pytest structure, fixtures/`conftest`, parametrized tables,
  fakes over mocks, `tmp_path`, coverage, and Hypothesis.
- **refactoring-python** — type hints + mypy, dataclasses, comprehensions/generators,
  `pathlib`, context managers, and fixing mutable defaults / bare excepts (ruff).
- **performance-python** — measure-first profiling with `cProfile`/`py-spy`/`timeit`,
  `tracemalloc`, the right fix (algorithm/caching/numpy), and the GIL escape hatch.
- **tiered-model-orchestration** — route a substantial task across model tiers:
  plan with Opus (max effort), execute with Sonnet, cross-verify with
  Haiku/Sonnet via an independent, adversarial pass.

## License

[Apache-2.0](../../LICENSE).
