---
name: debugging-python
description: Use when a Python program raises exceptions, hangs, blocks the event loop, or leaks memory — reading tracebacks and exception chains, pdb/post-mortem, py-spy, faulthandler, tracemalloc, and asyncio task inspection.
---

# Debugging Python

## Overview

**Observe before you theorize.** Each failure mode has a canonical tool that extracts ground truth from the process — reach for it before editing code: `py-spy dump` for a live hang, `faulthandler` when you own the code, `tracemalloc.compare_to` for leaks, and read the exception *chain*, not just the last line.

## Exceptions — read the traceback + follow the chain

Read the traceback: the **last line** is the type + message (what), the **frame above it** is the raise site (where), the frames above that are the path (how). Then follow the chain — the words *between* tracebacks are the causal story:
- `The above exception was the direct cause…` → explicit `raise X from Y`; **Y is the root cause**.
- `During handling of the above exception…` → a second exception fired inside an `except`/`finally` (often masking the real one); the **topmost** traceback is usually the root.

`logger.exception("...")` inside an `except` logs the full chained traceback. Never bare `except:` — use `except SomeError as e:` then `raise` (re-raise) or `raise New(...) from e` (preserve cause; `from None` to suppress). Post-mortem without re-running:
```bash
python -m pdb -c continue app.py     # drops into pdb at the unhandled exception
```
`import pdb; pdb.post_mortem()`; `breakpoint()` (honors `PYTHONBREAKPOINT`). At `(Pdb)`: `bt`, `up`/`down`, `pp obj`, `args`, `!code`.

## Hangs — dump the stuck stack

`top`/`htop` first for the shape: **~100% CPU** = busy loop / pathological regex; **~0% CPU** = blocked on I/O, a lock, or a wait. Then get the stack:
```bash
py-spy dump --pid 12345 --locals     # live process, no code change: every thread's current stack
py-spy top --pid 12345               # where wall-time goes, live
```
`py-spy dump` shows exactly which line each thread is parked on (a `recv()` with no timeout, a `lock.acquire()`); two threads each holding the other's lock = deadlock. `--native` unwinds C frames. Code-available equivalent, stdlib `faulthandler`:
```python
import faulthandler, signal
faulthandler.register(signal.SIGUSR1)          # kill -USR1 <pid> -> all-thread stacks to stderr
faulthandler.dump_traceback_later(30, repeat=True)  # watchdog: catch an intermittent hang in the act
```
`faulthandler` also survives a C-extension segfault (py-spy/pdb can't) — that tells you the bug is native.

## Memory leaks — measure allocations, attribute growth

A "leak" is usually reachable objects that only grow (an unbounded cache/list, a lingering global, an `lru_cache` with no `maxsize`, a captured traceback pinning frame locals). Confirm RSS trends up under steady load (`resource.getrusage(...).ru_maxrss`), then attribute:
```python
import tracemalloc; tracemalloc.start(25)
snap1 = tracemalloc.take_snapshot()
# ... run the leaky workload ...
for stat in tracemalloc.take_snapshot().compare_to(snap1, 'lineno')[:20]:
    print(stat)   # ranks by +size growth -> the exact file:line
```
Complement with object-type growth: `Counter(type(o).__name__ for o in gc.get_objects())` diffed over time, or `objgraph.show_growth()` + `objgraph.show_backrefs(...)` to see the reference chain holding it alive (that chain *is* the fix).

## asyncio — a blocked loop looks like a hang everywhere

- **Blocking call on the loop thread** (a `time.sleep`, `requests.get`, big CPU loop freezes *every* coroutine): `asyncio.run(main(), debug=True)` / `PYTHONASYNCIODEBUG=1` logs any callback exceeding `slow_callback_duration`, naming the culprit; `py-spy dump` shows the loop thread parked in the sync call (a healthy idle loop sits in `epoll_wait`). Fix: `await loop.run_in_executor(None, fn)` or an async driver.
- **Awaitable that never resolves:** `for t in asyncio.all_tasks(): t.print_stack()` shows the exact `await` each pending task is stuck on. Rule: every I/O await gets `asyncio.wait_for(...)`. Watch for "coroutine was never awaited" (a missing `await`) and `gather` swallowing a child exception (`return_exceptions=True` to surface it).

## Quick reference

| Symptom | First look | Tool |
|---|---|---|
| Exception | traceback bottom-up + `from`/"During handling" chain | `logger.exception`, `pdb.post_mortem`, `python -m pdb -c continue` |
| Hang (live) | `top`: 100% (busy) vs 0% (blocked) | `py-spy dump --pid --locals`, `py-spy top` |
| Hang (own code) | on-demand / watchdog stack | `faulthandler` + `kill -USR1`, `dump_traceback_later` |
| Async hang | blocked loop vs unresolved awaitable | debug mode + `slow_callback_duration`; `all_tasks()` + `print_stack()` |
| Leak | RSS trending up | `tracemalloc.compare_to`; `gc`/`objgraph` growth diff |

## Common mistakes

- Reading only the last traceback line, ignoring the `from`/"During handling" chain.
- `print` debugging a live/prod hang instead of `py-spy dump` (no code change, no restart).
- CPU profiler on a 0%-CPU hang — it's blocked, not computing; dump the stack.
- One heap guess instead of `tracemalloc.compare_to` between two snapshots.
- Blocking calls on the asyncio loop thread; I/O awaits with no `wait_for` timeout.
