---
name: howto-code-in-python
description: Use when writing, reviewing, or modifying Python code - covers uv project/package management, type checking, data modeling with dataclasses/attrs/Pydantic, async patterns, testing with pytest, and Ruff linting/formatting conventions
user-invocable: false
---

# Python House Style

## Overview

Python house style, built around `uv` as the project and package manager. Applies whenever writing, reviewing, or modifying Python code.

The two governing values: correctness over convenience, and pragmatic incrementalism. Type the boundaries aggressively, keep internal code simple, and let the toolchain (uv, Ruff, Pyright) do enforcement rather than convention alone.

## Quick Self-Check (Use Under Pressure)

When under deadline pressure or focused on other concerns, STOP and verify:

- [ ] Using `uv add` / `uv run` / `uv sync`, never raw `pip install`
- [ ] Type hints on all public function signatures: `X | Y` not `Optional`/`Union`, `list[int]` not `List[int]`
- [ ] `dataclass` or `attrs` for internal structures; Pydantic only at validated boundaries
- [ ] No mutable default arguments (`def f(x: list | None = None)`, never `def f(x: list = [])`)
- [ ] `raise ... from err` when wrapping/re-raising exceptions
- [ ] No bare `except:` (use `except Exception:` at minimum)
- [ ] `src/` layout, `tests/` as a sibling directory, not colocated
- [ ] `decimal.Decimal` for money, never `float`
- [ ] Config and env vars read only at entry points, never inside library code

**Why this matters:** Under pressure, muscle memory defaults to older Python idioms (`Optional[X]`, mutable defaults, bare except). These checks catch the most common regressions.

## Project and Package Management with uv

`uv` (Astral) is the standard tool for the full project lifecycle: environments, dependencies, Python versions, builds, and publishing. Do not use `pip`, `pipx`, `poetry`, or `pyenv` directly on a uv-managed project.

### Core commands

| Command | Purpose |
|---------|---------|
| `uv init` | Create a new project (`--lib` for a library, `--app` for an application) |
| `uv add <pkg>` | Add a dependency, updates `pyproject.toml` and `uv.lock` |
| `uv add --group dev <pkg>` | Add to a named dependency group (dev, test, lint, ...) |
| `uv remove <pkg>` | Remove a dependency |
| `uv sync` | Install the environment to match the lockfile exactly |
| `uv lock` | Re-resolve and update `uv.lock` |
| `uv run <cmd>` | Run a command inside the project's environment |
| `uv build` | Build sdist/wheel |
| `uv publish` | Upload to PyPI or a configured index |

`uv run pytest`, `uv run ruff check .`, etc. -- never activate a virtualenv manually and never assume one is already active. `uv run` guarantees the right interpreter and dependency set every time.

### Python version pinning

Pin the interpreter explicitly with `uv python pin 3.14`, which writes a `.python-version` file. `requires-python` in `pyproject.toml` must be compatible with the pin, or `uv sync` fails loudly rather than silently using the wrong interpreter. Don't rely on whatever Python happens to be on `PATH`.

**Target Python 3.14 as the floor for new projects.** Python 3.10 reaches end of life in October 2026 -- don't start new projects there. If a dependency's ecosystem support lags on 3.14, 3.13 is an acceptable fallback, but treat that as a documented exception, not the default.

### Dependency groups

Use PEP 735 `[dependency-groups]` in `pyproject.toml`, not the legacy `[tool.uv.dev-dependencies]` table:

```toml
[dependency-groups]
dev = ["ruff", "pyright"]
test = ["pytest>=8", "pytest-cov", "pytest-asyncio", "hypothesis", "syrupy"]
```

Run with `uv sync --group test` or `uv run --group test pytest`. PEP 735 groups are tool-agnostic -- pip and other tooling read the same table, unlike the uv-specific predecessor.

### Lockfile

**Always commit `uv.lock`**, for applications and libraries alike. It has no effect on how a published library resolves for downstream consumers (they resolve against your `pyproject.toml` version constraints, not your lockfile), but it makes your own development and CI reproducible. Use `uv sync --frozen` in CI to fail loudly if the lockfile is out of date rather than silently re-resolving.

### Workspaces

For monorepos with multiple packages, use `[tool.uv.workspace]` at the repo root:

```toml
[tool.uv.workspace]
members = ["packages/*"]
```

Workspace members share a single `.venv` and a single `uv.lock`. Reference sibling packages with `{ workspace = true }` in `[tool.uv.sources]`. All members must share a compatible `requires-python` range -- if two packages genuinely need different Python versions, they don't belong in the same workspace.

### Standalone tools

Use `uvx <tool>` (alias for `uv tool run`) for one-off/ephemeral tool invocations, and `uv tool install <tool>` for tools you want persistently on `PATH`. This replaces `pipx` entirely.

### Build backend

Default to `uv_build`:

```toml
[build-system]
requires = ["uv_build>=0.11.31,<0.12"]
build-backend = "uv_build"
```

It's fast and needs no configuration for ordinary pure-Python packages. Fall back to `hatchling` only when you need something `uv_build` doesn't do: C extensions, custom build hooks, or non-standard file inclusion rules.

**Dynamic versioning:** derive the package version from git tags rather than hardcoding it. Use `uv-dynamic-versioning` with `uv_build`, or `hatch-vcs` with `hatchling`. A `git tag vX.Y.Z && git push --tags` should be the only step required to cut a release.

### Dependency pinning strategy

uv's lockfile already gives you exact reproducibility, so `pyproject.toml` version constraints don't need to be exact pins the way Cargo's do.

**Applications:** loose constraints in `pyproject.toml` (`"pytest>=8"`), exact pins live in `uv.lock`. Let `uv lock --upgrade` handle bumps deliberately, reviewed via the lockfile diff.

**Libraries published to PyPI:** use the narrowest *range* that's actually correct (`"httpx>=0.27,<1"`), never an exact pin. Downstream consumers resolve against your declared range, not your lockfile -- an exact pin in a published library forces unnecessary version conflicts for anyone depending on it alongside another package with a different pin.

Verify current versions before adding or bumping a dependency -- use `ed3d-research-agents:internet-researcher` rather than a memorized version, which goes stale quickly.

## Type System

### Modern syntax

Use `X | Y`, never `Optional[X]` or `Union[X, Y]`. Use builtin generics (`list[int]`, `dict[str, int]`, `tuple[int, ...]`), never the deprecated `typing.List`/`typing.Dict`/`typing.Tuple`. Both are natively supported without `from __future__ import annotations` on the 3.14 floor this guide targets.

For generic classes and functions, use PEP 695 syntax:

```python
# GOOD: PEP 695 -- scoped type parameter, no module-level TypeVar
class Box[T]:
    def __init__(self, item: T) -> None:
        self.item = item

def first[T](items: list[T]) -> T:
    return items[0]

type Maybe[T] = T | None

# BAD: pre-3.12 TypeVar boilerplate for new code
from typing import TypeVar, Generic
T = TypeVar("T")
class Box(Generic[T]): ...
```

### Type checker

**Default to Pyright.** It has the highest conformance to the typing spec and the best editor integration (via Pylance), and its `strict` mode is the closest Python equivalent to TypeScript's `strict: true` baseline:

```json
{
  "typeCheckingMode": "strict"
}
```

Apply `strict` per-directory (`"strict": ["src"]`) rather than repo-wide on day one if adopting into an existing codebase -- raise the bar incrementally, module by module, not by suppressing errors wholesale.

**When to deviate:** if the codebase is large enough that check latency matters (hundreds of thousands of lines), `pyrefly` (Meta's Rust-based checker) is production-proven at that scale and worth adopting. Astral's own `ty` is the tool to watch for closest alignment with the rest of this uv+Ruff toolchain, but confirm it has left beta before making it the default -- don't adopt beta tooling as a project's primary gate. `mypy` remains fine for existing codebases already built around it, but don't choose it for new projects; it lags in conformance and speed.

### Protocol vs ABC

| Situation | Use | Why |
|-----------|-----|-----|
| Interface for a third-party or builtin type | `Protocol` | Can't add inheritance to code you don't own |
| New hierarchy you fully control | `ABC` | Runtime enforcement (`TypeError` on missing methods), can share default implementations |
| Static-only structural check, no runtime cost | `Protocol` | No coupling, no inheritance required |
| Need `__init_subclass__` validation or shared init logic | `ABC` | Protocols carry no runtime behavior |

```python
# GOOD: Protocol for structural typing against code you don't own
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

def cleanup(resource: SupportsClose) -> None:
    resource.close()

# GOOD: ABC for a hierarchy you own, with runtime enforcement
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def get(self, id: str) -> object: ...
```

## Data Modeling

Three tools, three distinct jobs. Don't default to Pydantic everywhere -- it's the heaviest of the three.

| Use case | Tool | Why |
|----------|------|-----|
| Internal value objects, config holders | `dataclasses` (stdlib) | Zero dependencies, fastest, no validation needed |
| Internal models that need validation or memory efficiency | `attrs` | Composable validators, `slots=True` cuts memory ~40%, faster than Pydantic |
| Validated boundary data (API payloads, external input, config from disk) | Pydantic v2 | Type coercion, validation, JSON schema generation |

```python
# GOOD: dataclass for an internal, already-trusted structure
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float

# GOOD: Pydantic for untrusted boundary data
from pydantic import BaseModel, EmailStr

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr
    age: int

# BAD: Pydantic for a purely internal, already-validated value
class Point(BaseModel):  # unnecessary validation overhead on the hot path
    x: float
    y: float
```

**Rule of thumb:** if the data crossed a trust boundary (network, disk, user input) and hasn't been validated yet, Pydantic. If it's constructed internally from already-valid data, `dataclass` or `attrs`.

## Error Handling Philosophy

Same two-tier model as the rest of this house style (see `coding-effectively`): user-facing errors get clear, actionable messages; internal/programming errors may propagate as unhandled exceptions with full tracebacks. Error messages are lowercase sentence fragments -- `failed to connect to database: connection refused`, not `Failed to Connect to Database`.

**Build a custom exception hierarchy** so callers can catch at the right granularity:

```python
class AppError(Exception):
    """Base for all app-specific errors."""

class ConfigurationError(AppError):
    """Configuration is invalid or missing."""

class UpstreamError(AppError):
    """A dependency we called failed."""
```

**Always chain exceptions when wrapping.** `raise X from err` preserves the original cause in the traceback instead of hiding it:

```python
# GOOD: cause preserved
try:
    conn = connect(dsn)
except OSError as err:
    raise UpstreamError(f"failed to connect to database: {dsn}") from err

# BAD: original traceback lost
try:
    conn = connect(dsn)
except OSError:
    raise UpstreamError("failed to connect to database")
```

**Never use bare `except:`.** It catches `KeyboardInterrupt` and `SystemExit` along with everything else, and it swallows bugs you didn't mean to catch. Use `except Exception:` as the outer bound, and prefer catching the specific exception type whenever you know it.

**Watch the except-block variable scope trap:** the name bound by `except X as e:` is deleted when the block exits. If you need the exception after the block, assign it to a different variable first.

```python
# BAD: `err` doesn't exist here -- Python deletes it at block exit
try:
    risky()
except ValueError as err:
    pass
print(err)  # NameError

# GOOD: capture what you need before the block ends
error_message = None
try:
    risky()
except ValueError as err:
    error_message = str(err)
```

## Async Patterns

Be selective with async, same as every other language in this house style: use it for I/O and concurrency, keep everything else synchronous.

- **Application code:** `asyncio.TaskGroup` (3.11+) for structured concurrency -- a nursery that awaits all child tasks and aggregates exceptions into an `ExceptionGroup`.
- **Library code:** use `anyio` instead of asyncio directly when the library should work under either asyncio or trio, and when you need capabilities `TaskGroup` doesn't provide: cancel scopes, task-readiness signaling, or a portable stream abstraction.
- **`trio`** is a fine reference implementation of structured concurrency but a niche direct dependency -- reach for it only if the project is specifically built around it, not by default.
- **Sync/async boundary:** isolate async to the modules that actually do I/O. Don't make a pure function `async def` just because it's called from async code -- call it synchronously from within the async function.

```python
# GOOD: structured concurrency, exceptions aggregated
async def fetch_all(urls: list[str]) -> list[Response]:
    results: list[Response] = []
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            tg.create_task(fetch_one(url, results))
    return results

# BAD: unnecessary async on a pure function
async def format_response(data: dict) -> str:  # no I/O, no reason to be async
    return json.dumps(data, indent=2)
```

## Testing

### Stack

| Purpose | Library |
|---------|---------|
| Test runner | `pytest` |
| Coverage | `pytest-cov` |
| Async tests | `pytest-asyncio` (`asyncio_mode = "auto"` in `[tool.pytest.ini_options]`) |
| Property-based testing | `hypothesis` -- see `property-based-testing` skill for when/how |
| Snapshot testing | `syrupy` -- `assert result == snapshot` |
| Parallel runs | `pytest-xdist` |

### Layout

`tests/` as a sibling of `src/`, not colocated with source files:

```
myproject/
  src/mypackage/
    module.py
  tests/
    conftest.py
    test_module.py
  pyproject.toml
```

Place fixtures at the lowest `conftest.py` that actually needs them, not all in the root `tests/conftest.py`. pytest discovers fixtures up the directory tree automatically, so a fixture only used by `tests/api/` belongs in `tests/api/conftest.py`.

### Never skip tests silently

If a test needs an environment variable, API key, or fixture that isn't available, it must **fail with a clear message**, not skip invisibly. `@pytest.mark.skipif` guarding on a missing external dependency is fine when the reason is explicit and logged; an early `return` that quietly turns a broken test into a passing one is not. A green suite must mean every test actually ran and verified something.

## Linting and Formatting

Ruff is the single tool for both linting and formatting -- it replaces Black, isort, flake8, and pyupgrade. Configure it in `pyproject.toml` under `[tool.ruff]`, consistent with keeping project config centralized rather than scattered across tool-specific files:

```toml
[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "SIM"]
```

That baseline covers: core errors (E/F/W), import ordering (I), automatic modernization (UP), common bug patterns (B, via flake8-bugbear), and simplification suggestions (SIM). Add `S` (flake8-bandit) for security-sensitive projects.

Use human-readable suppression comments, not bare codes: `# noqa: unused-import` rather than `# noqa: F401` -- self-documenting when someone reads it later.

Wire Ruff and Pyright into `pre-commit` (`.pre-commit-config.yaml`), including uv's own official hooks (`uv-lock`, `uv-export`) to keep the lockfile in sync automatically. If hook latency becomes a bottleneck, `prek` is a drop-in, faster replacement for the `pre-commit` runner itself.

## Module Organization

**src-layout, always**, for anything meant to be imported or distributed:

```
myproject/
  src/mypackage/
    __init__.py
    module.py
  tests/
  pyproject.toml
```

This forces tests and tooling to run against the *installed* package rather than accidentally importing the working-directory copy -- it catches packaging bugs (missing files, bad metadata) before they reach a release. Flat-layout (`mypackage/` at repo root, no `src/`) is an acceptable exception only for small, non-distributed internal scripts.

**Descriptive file names**, same principle as the rest of this house style: `user_validation.py` and `date_arithmetic.py`, never `utils.py` or `helpers.py`.

## Configuration and Environment

Library code must never read `os.environ` directly. Accept configuration as function arguments, dataclass fields, or constructor parameters. Environment variable reading belongs exclusively at application entry points -- `main()`, CLI argument parsing, or a dedicated settings loader (Pydantic's `BaseSettings` is the standard tool for that specific job, at the entry point, not scattered through library internals).

This is also just FCIS: `os.environ` access is I/O, so it can never live in a Functional Core file regardless of this rule -- the rule exists to make the boundary explicit even in Imperative Shell code that could technically read env vars anywhere.

## Research Before Guessing

When you hit an unfamiliar package, an ambiguous API, or a dependency version question, use `ed3d-research-agents:internet-researcher` rather than iterating by trial and error or relying on memorized (and likely stale) API shapes. This runs in isolated context and returns a summary -- it doesn't pollute working context the way speculative trial-and-error does.

## Sharp Edges

Runtime hazards the type checker won't catch. Know these cold.

### Mutable default arguments

Default argument values are evaluated **once**, at function definition time, and reused across every call.

```python
# BAD: the same list is shared and mutated across every call
def add_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items

# GOOD: sentinel default, fresh list per call
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items
```

### Late-binding closures in loops

Closures capture the *variable*, not its value at closure-creation time.

```python
# BAD: every lambda captures the same `i`, all print 2
callbacks = [lambda: i for i in range(3)]
[cb() for cb in callbacks]  # [2, 2, 2]

# GOOD: default argument captures the value at definition time
callbacks = [lambda i=i: i for i in range(3)]
[cb() for cb in callbacks]  # [0, 1, 2]
```

### `is` vs `==`

`is` compares identity, not equality. Small integers (-5 to 256) and short strings are cached by CPython, which makes `is` appear to "work" by accident -- don't rely on it.

```python
# BAD: relies on CPython implementation detail, breaks for larger ints
a = 1000
b = 1000
a is b  # False (usually) -- not guaranteed by the language

# GOOD: always use == for value comparison
a == b  # True

# `is` is correct only for singleton checks
value is None
value is True
```

### Decimal for money

`float` is IEEE 754 binary floating point -- it cannot represent most decimal fractions exactly.

```python
# BAD: precision errors compound
0.1 + 0.2  # 0.30000000000000004

# GOOD: exact decimal arithmetic
from decimal import Decimal
Decimal("0.1") + Decimal("0.2")  # Decimal('0.3')
```

Always construct `Decimal` from a string or integer, never from a `float` literal (`Decimal(0.1)` inherits the float's imprecision before conversion).

### Shallow vs deep copy

`copy.copy()` and slicing (`list[:]`) copy one level deep -- nested mutable objects are still shared.

```python
import copy

original = {"items": [1, 2, 3]}
shallow = copy.copy(original)
shallow["items"].append(4)
original["items"]  # [1, 2, 3, 4] -- mutated through the shallow copy

deep = copy.deepcopy(original)
deep["items"].append(5)
original["items"]  # unaffected
```

### f-strings and injection

f-strings are the standard for string formatting -- but never build SQL, shell commands, or other executable content by interpolating untrusted input into one.

```python
# DANGEROUS: SQL injection
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# SAFE: parameterized query
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
```

### Circular imports

Python resolves imports at module load time; two modules importing each other at the top level will fail unpredictably depending on which one loads first. Break the cycle by moving the shared dependency to a third module, or by importing inside the function that needs it (a last resort, not a pattern).

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Optional[X]` / `Union[X, Y]` | Use `X | None` / `X | Y` |
| `List[int]`, `Dict[str, int]` | Use `list[int]`, `dict[str, int]` |
| Mutable default argument (`def f(x=[])`) | Default to `None`, initialize inside the function |
| Bare `except:` | `except Exception:` at minimum, prefer specific types |
| `raise NewError("...")` inside an `except` block | `raise NewError("...") from err` |
| `float` for money | `decimal.Decimal` |
| Pydantic for purely internal structures | `dataclass` or `attrs` |
| `os.environ` read inside library code | Accept config as parameters; read env only at entry points |
| Colocated tests next to source | `tests/` as a sibling of `src/` |
| Flat project layout for a distributed package | `src/` layout |
| `pip install` in a uv project | `uv add` / `uv run` |
| f-string building SQL/shell commands | Parameterized queries / `subprocess` argument lists |
| Test silently skips on missing fixture/env var | Fail loudly with a clear message |

## Red Flags

**STOP and refactor when you see:**

- `typing.List`/`Dict`/`Optional`/`Union` in new code
- A mutable object (`list`, `dict`, `set`) as a default argument value
- Bare `except:` or an `except Exception:` that swallows the error silently
- `float` used for currency or financial calculations
- `os.environ.get(...)` inside a library module, not at the entry point
- Pydantic `BaseModel` used for data that never crosses a trust boundary
- A package installed with `pip install` instead of `uv add` in a uv-managed project
- Tests colocated with source, or a project without a `src/` layout
- A closure created inside a loop without capturing the loop variable via a default argument
