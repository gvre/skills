---
name: python-developer
description:
  Comprehensive rules and best practices for writing modern Python 3.12+ projects.
  Use when creating new Python projects, reviewing Python code, refactoring existing code,
  or when the user asks about Python project structure, dependencies, type hints, testing, or modern Python features.
  Ensures code follows current best practices with proper tooling (uv, ruff, mypy, pytest), type hints, dataclasses,
  async patterns, and project structure. Includes version-specific features for 3.12, 3.13, and 3.14.
disable-model-invocation: true
license: MIT
metadata:
  author: Giannis Vrentzos
  version: "1.0.0"
---

# Python Developer

Guide for writing production-quality Python 3.12+ projects following current best practices.

## Version Support

This guide covers:
- **Python 3.12+**: Base requirements and common features (all sections below)
- **Python 3.13**: Additional features and improvements → See [references/python-3.13-features.md](references/python-3.13-features.md)
- **Python 3.14**: Latest features and enhancements → See [references/python-3.14-features.md](references/python-3.14-features.md)

### Feature Version Matrix

| Feature | 3.12 | 3.13 | 3.14 |
|---------|------|------|------|
| Type parameter syntax (`def func[T]()`) | ✅ | ✅ | ✅ |
| Type statement (`type X = ...`) | ✅ | ✅ | ✅ |
| Pattern matching (`match/case`) | ✅ | ✅ | ✅ |
| Exception groups (`except*`) | ✅ | ✅ | ✅ |
| TaskGroup | ✅ | ✅ | ✅ |
| TypeIs for type narrowing | ❌ | ✅ | ✅ |
| ReadOnly type hint | ❌ | ✅ | ✅ |
| Improved REPL | ❌ | ✅ | ✅ |
| Experimental JIT | ❌ | ❌ | ✅ |
| Free-threaded (no-GIL) | ❌ | ❌ | ✅ |

## Quick Reference

**Core Requirements (3.12+):**
- Python 3.12+ minimum
- Use pyproject.toml for all config
- Type hints everywhere
- Ruff for formatting/linting
- mypy or ty for type checking
- pytest for testing
- src/ layout structure

**Type Checker Comparison:**

| Feature | mypy | ty |
|---------|------|-----|
| Status | ✅ Stable, mature (1.0+) | ⚠️ Beta (0.0.5, Dec 2025) |
| Speed | Standard | 10-100x faster |
| Language | Python | Rust |
| Ecosystem | Extensive | Growing (very new) |
| IDE Support | Universal | VS Code, PyCharm, Neovim |
| Production Ready | ✅ Yes | ⚠️ Not yet - expect changes |
| Best for | Most projects, production systems | Experimentation, speed testing |
| Maintainer | Python community | Astral (Ruff/uv creators) |
| Configuration | `[tool.mypy]` | `[tool.ty]` |
| Recommendation | **Default choice** | Try for new projects only |

## Project Structure

```
project-name/
├── src/
│   └── package_name/
│       ├── __init__.py
│       └── core.py
├── tests/
│   └── test_core.py
├── pyproject.toml
└── README.md
```

**Rules:**
- ALWAYS use src/ layout (never place code at project root)
- ALWAYS include __init__.py in packages
- Use underscores in module names: `my_module.py` not `my-module.py`

## Configuration (pyproject.toml)

### Base Configuration (3.12+)

```toml
[project]
name = "your-project"
version = "0.1.0"
description = "Your description"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.28.1",
]

[project.optional-dependencies]
dev = [
    "ruff>=0.14.10",
    "mypy>=1.19.1",
]
test = [
    "pytest>=9.0.2",
    "pytest-asyncio>=1.3.0",
    "pytest-cov>=7.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "C4", "UP"]

[tool.mypy]
python_version = "3.12"
strict = true
```

## Type Hints (Python 3.12+)

**Required everywhere:**
- All function parameters and return types
- Class attributes
- Module-level variables when not obvious

**Modern syntax (3.10+):**
```python
# Use | for unions (not Optional/Union)
def process(data: str | None) -> dict[str, int | float]:
    ...

# Use lowercase generics (not List/Dict)
def filter_items(items: list[str]) -> list[str]:
    ...

# Use collections.abc for abstract types
from collections.abc import Sequence, Mapping, Iterable

def process_items(items: Sequence[str]) -> Iterable[str]:
    """Process items - accepts list, tuple, or any sequence."""
    return (item.upper() for item in items)
```

**Python 3.12+ Type Features (PEP 695):**
```python
# Type parameters with new syntax
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

# Type statement for aliases
type Point = tuple[float, float]

# Generic classes
class Container[T]:
    def __init__(self, value: T) -> None:
        self.value = value

    def get(self) -> T:
        return self.value
```

**Advanced patterns:** See [references/core-language-features.md](references/core-language-features.md) for comprehensive type hint patterns, Protocol, TypedDict, Literal, and overload examples.

**Forbidden:**
- NEVER import from typing: `List`, `Dict`, `Tuple`, `Optional`, `Union`
- NEVER leave functions untyped

## Modern Features

### Dataclasses (Python 3.7+, slots from 3.10+)

```python
from dataclasses import dataclass
from typing import Self

@dataclass(slots=True)
class User:
    id: int
    name: str
    email: str | None = None

    def with_email(self, email: str) -> Self:
        return User(id=self.id, name=self.name, email=email)
```

### Pattern Matching - Basic (Python 3.10+)

```python
def handle_command(cmd: str) -> str:
    match cmd:
        case "quit" | "exit":
            return "Exiting"
        case "help":
            return "Showing help"
        case _:
            return f"Unknown: {cmd}"
```

### Functools Patterns

```python
from functools import cache, lru_cache

@cache  # Unbounded cache for pure functions
def expensive_computation(n: int) -> int:
    return n ** 2

@lru_cache(maxsize=128)  # Limited cache
def fetch_data(url: str) -> dict[str, Any]:
    ...
```

### Pathlib (not os.path)

```python
from pathlib import Path

config_dir = Path.home() / ".config" / "myapp"
config_file = config_dir / "config.json"
```

### Context Managers

```python
import os
from collections.abc import Iterator
from contextlib import contextmanager

@contextmanager
def temporary_setting(name: str, value: str) -> Iterator[None]:
    old_value = os.getenv(name)
    os.environ[name] = value
    try:
        yield
    finally:
        if old_value is None:
            del os.environ[name]
        else:
            os.environ[name] = old_value
```

## Async Programming

### Basic Async (All versions)

```python
import asyncio
import httpx

async def fetch_url(url: str) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text
```

### Task Groups (Python 3.11+)

```python
async def fetch_all(urls: list[str]) -> list[str]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_url(url)) for url in urls]
    return [task.result() for task in tasks]
```

**Advanced patterns:** See [references/core-language-features.md](references/core-language-features.md) for decorators, descriptors, walrus operator, context variables, advanced pattern matching, and metaclasses.

## Testing (pytest)

**Key patterns:**
- Use pytest (never unittest)
- Parametrize to test multiple scenarios
- Mock external dependencies
- Use fixtures for setup/teardown
- Mark async tests with `@pytest.mark.asyncio`

See [references/testing-guide.md](references/testing-guide.md) for comprehensive testing patterns, async testing, mocking, fixtures, and coverage configuration.

## Code Quality

### Ruff Configuration

```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "C4", "UP"]
```

### Type Checking

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

### Naming Conventions

- **Functions/variables:** snake_case
- **Classes:** PascalCase
- **Constants:** UPPER_SNAKE_CASE
- **Private:** prefix with `_`

✅ GOOD: `calculate_average()`, `UserManager`, `MAX_RETRIES`
❌ BAD: `CalculateAverage()`, `calc_avg()`, `userManager`

## Error Handling

**Standard exception handling (All versions):**
```python
try:
    risky_operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
except TimeoutError:
    logger.warning("Operation timed out")
```

**Catching `Exception` - when it's acceptable:**
```python
# ✅ Top-level handler (API endpoint, CLI) - handle gracefully
@app.route("/api/data")
def get_data():
    try:
        return process_data()
    except Exception as e:
        logger.exception("Request failed")
        return {"error": "Internal server error"}, 500

# ✅ Batch processing - log and continue
for item in items:
    try:
        process(item)
    except Exception as e:
        logger.exception(f"Failed to process {item}")
        continue

# ✅ Add context and wrap
except Exception as e:
    raise ProcessingError(f"Failed for {item_id}") from e

# ❌ Never silently swallow
except Exception:
    pass
```

**Exception groups (Python 3.11+):**
```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(task1())
        tg.create_task(task2())
except ExceptionGroup as eg:
    for exc in eg.exceptions:
        if isinstance(exc, ValueError):
            handle_value_error(exc)
        else:
            raise
```

## Logging

**Basic setup:**
```python
import logging
from pathlib import Path

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(name)s - %(message)s",
)
```

**Production best practices:** See [references/logging-guide.md](references/logging-guide.md) for structured logging (structlog vs python-json-logger), async logging, correlation IDs, request tracking, and performance optimization.

## Dependencies

```toml
[project]
dependencies = [
    "httpx>=0.28.1",
    "pydantic>=2.5.0",
]

[project.optional-dependencies]
dev = [
    "ruff>=0.14.10",
    "mypy>=1.19.1",
]
test = [
    "pytest>=9.0.2",
    "pytest-asyncio>=1.3.0",
    "pytest-cov>=7.0.0",
]
```

## Anti-Patterns (Never Do)

- ❌ Mutable default arguments: `def func(items=[])`
- ❌ Bare `except:` (catches SystemExit, KeyboardInterrupt)
- ❌ Silently swallowing exceptions: `except Exception: pass`
- ❌ Using eval/exec on user input
- ❌ Using `os.path` (use pathlib)
- ❌ `from typing import List, Dict` (use built-in generics)
- ❌ Hardcoded secrets or API keys
- ❌ Unparameterized SQL queries

## Documentation

```python
def calculate_average(
    numbers: list[float],
    *,
    weights: list[float] | None = None
) -> float:
    """Calculate the average of numbers.

    Args:
        numbers: List of numbers to average
        weights: Optional weights for weighted average

    Returns:
        The calculated average

    Raises:
        ValueError: If numbers is empty

    Examples:
        >>> calculate_average([1, 2, 3])
        2.0
    """
```

**Rules:** Google or NumPy docstring style. Include docstrings for public APIs. Types in signature, not docstring.

## Python 3.13 and 3.14 Features

- **Python 3.13**: See [references/python-3.13-features.md](references/python-3.13-features.md)
- **Python 3.14**: See [references/python-3.14-features.md](references/python-3.14-features.md)

## Security Best Practices

**Critical rules:**
- ✅ ALWAYS validate and sanitize user input
- ✅ ALWAYS use parameterized queries for SQL
- ✅ ALWAYS hash passwords (Argon2 or bcrypt)
- ✅ ALWAYS load secrets from environment
- ✅ ALWAYS use HTTPS for external APIs
- ❌ NEVER hardcode secrets
- ❌ NEVER use string concatenation for SQL
- ❌ NEVER log sensitive data

See [references/security-guide.md](references/security-guide.md) for input validation, SQL injection prevention, path traversal prevention, password hashing, secrets management, and comprehensive security patterns.

## Checklist

### For Code Authors

When creating Python code:

- [ ] Python 3.12+ features used
- [ ] All functions have type hints
- [ ] Modern type syntax used (`|`, lowercase generics, `collections.abc`)
- [ ] Dataclasses with `slots=True` for data
- [ ] Pathlib instead of os.path
- [ ] F-strings for formatting
- [ ] Specific exceptions caught
- [ ] Async with proper error handling and timeouts
- [ ] Tests written (pytest) with mocking
- [ ] Security best practices followed
- [ ] Docstrings for public APIs
- [ ] No hardcoded secrets
- [ ] Ruff and type checker (mypy/ty) pass
- [ ] src/ layout used

### For Code Reviewers

When reviewing Python code, prioritize findings:

**Priority 1 - Critical Issues (Must Fix):**
1. **Security vulnerabilities** (SQL injection, hardcoded secrets, path traversal, unsafe eval)
2. **Type safety violations** (missing type hints, forbidden typing imports, bare except)
3. **Correctness issues** (mutable defaults, async errors, missing timeouts)

**Priority 2 - Important Improvements (Should Fix):**
1. **Modern Python adoption** (os.path vs pathlib, old-style formatting)
2. **Error handling** (overly broad exceptions, missing specificity)
3. **Resource management** (missing context managers)
4. **Testing gaps** (missing tests, low coverage <80%, not mocking)

**Priority 3 - Nice to Have (Consider Fixing):**
1. **Documentation** (missing/incomplete docstrings)
2. **Code quality** (naming conventions, could use pattern matching)
3. **Performance** (missing @cache, could use TaskGroups)

## Resources

- **Full reference:** See [references/REFERENCE.md](references/REFERENCE.md)
- **Core language features:** See [references/core-language-features.md](references/core-language-features.md)
- **Logging guide:** See [references/logging-guide.md](references/logging-guide.md)
- **Testing guide:** See [references/testing-guide.md](references/testing-guide.md)
- **Security guide:** See [references/security-guide.md](references/security-guide.md)
- **Python 3.13 features:** See [references/python-3.13-features.md](references/python-3.13-features.md)
- **Python 3.14 features:** See [references/python-3.14-features.md](references/python-3.14-features.md)
