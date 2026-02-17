# Python Project Rules for Claude

These rules govern how Claude should write, review, and work with Python code in projects targeting Python 3.12+.

---

## Core Requirements

### Python Version
- **ALWAYS** target Python 3.12 as minimum version
- Use `requires-python = ">=3.12"` in pyproject.toml
- Leverage Python 3.12+ features without backwards compatibility compromises
- Set `python_version = "3.12"` in all tool configurations

**Version Support:**
- **Python 3.12+**: Base requirements (minimum target)
- **Python 3.13**: Additional type system features, improved REPL, performance improvements
- **Python 3.14**: Experimental JIT, free-threaded mode (officially supported)

**Feature Availability Matrix:**

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

**Python 3.13 Key Features:**
```python
from typing import TypeIs, ReadOnly

# TypeIs for better type narrowing
def is_str_list(val: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(x, str) for x in val)

# ReadOnly for immutable type hints
type Config = ReadOnly[dict[str, str]]
```

**Python 3.14 Key Features:**
- Experimental JIT compiler (enable with `PYTHON_JIT=1`)
- Free-threaded mode (enable with `PYTHON_GIL=0`, officially supported)
- JIT is experimental; free-threaded mode is production-ready with compatible libraries

### Project Structure
- **ALWAYS** use src layout:
  ```
  project/
  ├── src/
  │   └── package_name/
  ├── tests/
  ├── docs/
  ├── pyproject.toml
  └── README.md
  ```
- **NEVER** place package code at project root
- **ALWAYS** include __init__.py files in packages
- Use underscores in package/module names, not hyphens

### Configuration
- **ALWAYS** use pyproject.toml for all configuration
- **NEVER** create setup.py, setup.cfg, or requirements.txt for new projects
- Consolidate ALL tool config (ruff, ty, pytest) in pyproject.toml
- Include .python-version file with "3.12" for version managers

---

## Type Hints

### Required Usage
- **ALWAYS** include type hints for:
  - All function parameters
  - All function return types
  - Class attributes
  - Module-level variables when not obvious
- **NEVER** use bare `except:` or untyped functions in production code

### Forward References
- Use string quotes for forward references when needed:
  ```python
  from myapp.models import User, Organization
  from collections.abc import Sequence, Callable
  
  # Use quotes for forward references
  def process(user: User) -> Organization:
      return user.org
  
  def filter_users(users: Sequence[User]) -> list[User]:
      return [u for u in users if u.active]
  ```

### TYPE_CHECKING - Use for Import Optimization
- **USE** `TYPE_CHECKING` only for:
  - Heavy dependencies to avoid runtime import overhead:
    ```python
    from typing import TYPE_CHECKING
    
    if TYPE_CHECKING:
        import pandas as pd
        import tensorflow as tf
    
    def analyze(df: "pd.DataFrame") -> None:
        """pandas only imported by type checker."""
        pass
    ```
  - When you need `isinstance()` checks with imported types:
    ```python
    from typing import TYPE_CHECKING
    
    if TYPE_CHECKING:
        from myapp.models import User
    else:
        from myapp.models import User  # Import at runtime for isinstance()
    
    def process(data: User | dict) -> None:
        if isinstance(data, User):  # Works because User imported at runtime
            handle_user(data)
    ```
- **DO NOT** use `TYPE_CHECKING` for:
  - Standard library imports (always fast to import)
  - Regular application imports without circular dependency issues

### Modern Syntax (Python 3.12+)
- **ALWAYS** use `|` for unions: `str | None` not `Optional[str]`
- **ALWAYS** use lowercase generics: `list[str]` not `List[str]`
- **ALWAYS** use `collections.abc` types for parameters: `Sequence`, `Mapping`, `Iterable`, `Callable`
- **ALWAYS** use PEP 695 type parameters when appropriate:
  ```python
  def first[T](items: list[T]) -> T | None:
      return items[0] if items else None
  ```
- **ALWAYS** use `type` statement for type aliases:
  ```python
  type Point = tuple[float, float]
  type JSONDict = dict[str, Any]
  ```
- Use `Self` from typing for return types in methods that return instances
- Use `collections.abc` types for flexibility:
  ```python
  from collections.abc import Sequence, Mapping, Iterable, Callable

  def process_items(items: Sequence[str]) -> Iterable[str]:
      """Accepts list, tuple, or any sequence."""
      return (item.upper() for item in items)

  def transform(func: Callable[[int], str], data: Mapping[str, int]) -> list[str]:
      """More flexible than dict - accepts any mapping."""
      return [func(v) for v in data.values()]
  ```

### Advanced Type Patterns
- **USE** Protocol for structural subtyping (duck typing with types):
  ```python
  from typing import Protocol

  class Drawable(Protocol):
      def draw(self) -> None: ...

  class Circle:
      def draw(self) -> None:
          print("Drawing circle")

  def render(obj: Drawable) -> None:  # Accepts any object with draw()
      obj.draw()
  ```
- **USE** TypedDict for structured dictionaries:
  ```python
  from typing import TypedDict

  class UserDict(TypedDict):
      id: int
      name: str
      email: str | None

  def create_user(data: UserDict) -> User:
      return User(**data)
  ```
- **USE** Literal for restricted values:
  ```python
  from typing import Literal

  Status = Literal["pending", "active", "completed", "cancelled"]

  def update_status(status: Status) -> None:
      """Only accepts the specific literal values."""
      ...
  ```
- **USE** overload for multiple signatures:
  ```python
  from typing import overload

  @overload
  def process(data: str) -> str: ...

  @overload
  def process(data: int) -> int: ...

  def process(data: str | int) -> str | int:
      """Process returns same type as input."""
      return data
  ```

### Prohibited Type Hint Patterns
- **NEVER** use: `from typing import List, Dict, Tuple, Optional, Union`
- **NEVER** use: `typing.List`, `typing.Dict`, etc. (use built-in generics)
- **NEVER** leave function signatures untyped
- Avoid `Any` unless truly necessary (document why if used)
- **PREFER** `collections.abc` types over `typing` equivalents for parameters

---

## Code Style

### Formatting
- **ALWAYS** use Ruff for formatting and linting
- Line length: 100 characters
- Use double quotes for strings
- Use trailing commas in multi-line structures
- **NEVER** manually format code - let Ruff handle it

### Naming Conventions
- **snake_case** for: functions, methods, variables, modules
- **PascalCase** for: classes, type aliases, exceptions
- **UPPER_SNAKE_CASE** for: constants
- **NEVER** use single-letter variable names except:
  - Loop counters: `i`, `j`, `k` (when intent is clear)
  - Coordinates: `x`, `y`, `z`
  - Generic type variables: `T`, `K`, `V`
- Prefix private attributes/methods with single underscore: `_internal_method`

### Import Rules
- **ALWAYS** organize imports in order: stdlib, third-party, local
- **ALWAYS** use absolute imports for your own package
- **NEVER** use wildcard imports: `from module import *`
- **NEVER** use relative imports outside package
- Use Ruff's isort integration for automatic sorting

---

## Modern Python Features

### Core Language Features

**Walrus Operator (`:=`) - Assignment Expressions:**
- **USE** to eliminate redundant computations
- **USE** in comprehensions and while loops
- **DON'T OVERUSE** - readability matters

```python
# Good: avoid redundant function calls
if (result := expensive_operation()) is not None:
    process(result)

# Good: in comprehensions
data = [y for x in items if (y := transform(x)) is not None]

# Good: in while loops
while (line := file.readline()):
    process(line)

# Bad: overusing for simple cases
if (x := 5) > 3:  # Just use x = 5; if x > 3:
    pass
```

**Context Variables (`contextvars`) - For Async Context:**
- **ALWAYS** use for request-scoped data in async applications
- **USE** for correlation IDs, user context, transaction IDs
- **NEVER** use thread-local storage in async code

```python
from contextvars import ContextVar

# Define context variables
request_id: ContextVar[str] = ContextVar("request_id", default="")
user_id: ContextVar[str | None] = ContextVar("user_id", default=None)

async def handle_request(req_id: str, uid: str) -> None:
    request_id.set(req_id)
    user_id.set(uid)
    await process_request()  # Context available in nested calls

async def process_request() -> None:
    # Access context from any depth
    logger.info(f"Processing request {request_id.get()} for user {user_id.get()}")
```

**Descriptors - For Advanced Property Patterns:**
- **USE** for reusable validation logic
- **USE** for computed properties with caching
- **CONSIDER** before writing repetitive property code

```python
class ValidatedString:
    def __init__(self, min_length: int = 0, max_length: int = 100) -> None:
        self.min_length = min_length
        self.max_length = max_length
        self.name = ""
    
    def __set_name__(self, owner: type, name: str) -> None:
        self.name = f"_{name}"
    
    def __get__(self, obj: object, objtype: type | None = None) -> str:
        return getattr(obj, self.name, "")
    
    def __set__(self, obj: object, value: str) -> None:
        if not isinstance(value, str):
            raise TypeError(f"{self.name} must be a string")
        if not self.min_length <= len(value) <= self.max_length:
            raise ValueError(f"{self.name} length must be between {self.min_length} and {self.max_length}")
        setattr(obj, self.name, value)

class User:
    username = ValidatedString(min_length=3, max_length=20)
    email = ValidatedString(min_length=5, max_length=100)
```

**Metaclasses - Use Sparingly:**
- **RARELY** use - only for framework-level code
- **PREFER** class decorators or `__init_subclass__` instead
- **NEVER** use for simple class customization

```python
# Good: Use __init_subclass__ instead of metaclass
class Plugin:
    plugins: dict[str, type[Plugin]] = {}
    
    def __init_subclass__(cls, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        cls.plugins[cls.__name__] = cls

# Only use metaclass for truly meta operations (rare)
class SingletonMeta(type):
    _instances: dict[type, object] = {}
    
    def __call__(cls, *args: object, **kwargs: object) -> object:
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

### Iterators and Generators

**Use generators for memory efficiency:**

```python
# Good: generator for large sequences
def read_large_file(path: Path) -> Iterator[str]:
    with path.open() as f:
        for line in f:
            yield line.strip()

# Bad: loads entire file into memory
def read_large_file_bad(path: Path) -> list[str]:
    with path.open() as f:
        return [line.strip() for line in f]
```

**Rules:**
- **ALWAYS** use generators for large or infinite sequences
- **USE** `yield` instead of building lists when processing items one at a time
- **USE** generator expressions over list comprehensions when immediate materialization isn't needed
- **PREFER** `Iterator` or `Iterable` type hints for function parameters accepting sequences

```python
from collections.abc import Iterator, Iterable

# Good: accepts any iterable, returns generator
def process_items(items: Iterable[str]) -> Iterator[str]:
    for item in items:
        yield item.upper()

# Good: generator expression
processed = (x * 2 for x in range(1000000))

# Bad: unnecessary list
processed = [x * 2 for x in range(1000000)]  # Uses lots of memory
```

### Decorators

**Use decorators for cross-cutting concerns:**

```python
from functools import wraps
from time import perf_counter
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")

def timer(func: Callable[P, R]) -> Callable[P, R]:
    """Measure function execution time."""
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = perf_counter()
        result = func(*args, **kwargs)
        elapsed = perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def expensive_operation() -> int:
    return sum(range(1000000))
```

**Rules:**
- **USE** decorators for logging, timing, caching, validation
- **ALWAYS** use `@wraps` to preserve function metadata
- **USE** `ParamSpec` and `TypeVar` for proper decorator typing
- **PREFER** built-in decorators (`@property`, `@staticmethod`, `@classmethod`)
- **USE** `@functools.cache` or `@functools.lru_cache` for memoization

```python
from functools import cache, lru_cache

@cache  # Unlimited cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

@lru_cache(maxsize=128)  # Limited cache
def fetch_user(user_id: int) -> User:
    return db.query(User).get(user_id)
```

### Dataclasses
- **ALWAYS** prefer dataclasses over regular classes for data containers
- **ALWAYS** use slots for memory efficiency: `@dataclass(slots=True)`
- Use `frozen=True` for immutable data
- Example:
  ```python
  from dataclasses import dataclass

  @dataclass(frozen=True, slots=True)
  class User:
      id: int
      name: str
      email: str | None = None
  ```

### Pattern Matching
- **USE** structural pattern matching for complex conditionals:
  ```python
  match value:
      case {"type": "user", "id": user_id}:
          handle_user(user_id)
      case {"type": "admin", **rest}:
          handle_admin(rest)
      case _:
          handle_unknown()
  ```
- **PREFER** pattern matching over long if/elif chains with dict checks

### Path Handling
- **ALWAYS** use `pathlib.Path` for file operations
- **NEVER** use `os.path` for new code
- Example:
  ```python
  from pathlib import Path

  config_dir = Path("config")
  config_file = config_dir / "settings.yaml"

  if config_file.exists():
      content = config_file.read_text()
  ```

### Context Managers
- **ALWAYS** use context managers for resources:
  - File operations
  - Network connections
  - Database connections
  - Locks and semaphores
- Create custom context managers with `@contextmanager` when needed

### Functools Patterns

**Always use appropriate functools decorators:**

```python
from functools import lru_cache, cache, cached_property

# Use @cache for expensive pure functions (Python 3.9+)
@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Use @lru_cache when you need size limits
@lru_cache(maxsize=128)
def fetch_user(user_id: int) -> User:
    return expensive_db_query(user_id)

# Use @cached_property for expensive property computations
class DataProcessor:
    @cached_property
    def expensive_result(self) -> dict[str, object]:
        return compute_expensive_thing()
```

**Rules:**
- **USE** `@cache` for unlimited pure function memoization
- **USE** `@lru_cache(maxsize=N)` when memory limits matter
- **USE** `@cached_property` for lazy-loaded expensive properties
- **NEVER** cache functions with side effects

### F-strings
- **ALWAYS** use f-strings for string formatting
- **NEVER** use `%` formatting or `.format()` for new code
- Use `=` specifier for debugging: `f"{variable=}"`
- Example:
  ```python
  name = "Alice"
  age = 30
  print(f"{name} is {age} years old")
  print(f"{name=}, {age=}")  # Debug output
  ```

---

## Error Handling

### Exception Best Practices
- **ALWAYS** catch specific exceptions, never bare `except:`
- **ALWAYS** include exception context when re-raising
- Use exception groups for handling multiple errors:
  ```python
  try:
      async with asyncio.TaskGroup() as tg:
          tg.create_task(task1())
          tg.create_task(task2())
  except* ValueError as eg:
      for exc in eg.exceptions:
          log.error(f"ValueError: {exc}")
  ```
- Create custom exceptions that inherit from appropriate base classes
- **NEVER** use exceptions for flow control

### Validation
- Use Pydantic for data validation at boundaries (API, config, etc.)
- Use assertions for invariants and development checks
- **NEVER** use assertions for data validation in production code

---

## Async Programming

### When to Use Async
- **USE** async for I/O-bound operations:
  - HTTP requests
  - Database queries
  - File I/O (when using aiofiles)
  - Network operations
- **DON'T** use async for CPU-bound operations (use multiprocessing)

### Async Patterns
- **ALWAYS** use `async with` for async context managers
- **ALWAYS** use `asyncio.gather()` for concurrent operations
- **PREFER** `asyncio.TaskGroup()` (Python 3.11+) over `gather()`
- **ALWAYS** handle errors in async operations
- **ALWAYS** set timeouts for external requests
- Use `asyncio.create_task()` for fire-and-forget operations
- Example with error handling and timeouts:
  ```python
  import asyncio
  import httpx
  from collections.abc import Sequence

  async def fetch_url(url: str, client: httpx.AsyncClient) -> str:
      """Fetch a single URL with error handling."""
      try:
          response = await client.get(url, timeout=10.0)
          response.raise_for_status()
          return response.text
      except httpx.HTTPError as e:
          # Log error but don't crash - return error message
          return f"Error fetching {url}: {e}"

  async def fetch_all(urls: Sequence[str]) -> list[str]:
      """Fetch multiple URLs concurrently with shared client."""
      async with httpx.AsyncClient() as client:
          async with asyncio.TaskGroup() as tg:
              tasks = [tg.create_task(fetch_url(url, client)) for url in urls]
      return [task.result() for task in tasks]

  async def fetch_all_with_timeout(
      urls: Sequence[str],
      timeout_seconds: float = 30.0
  ) -> list[str]:
      """Fetch all URLs with a total timeout."""
      try:
          return await asyncio.wait_for(fetch_all(urls), timeout=timeout_seconds)
      except asyncio.TimeoutError:
          return [f"Timeout after {timeout_seconds}s" for _ in urls]
  ```

### Async Context Managers

**Always implement proper async resource management:**

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

@asynccontextmanager
async def database_connection(url: str) -> AsyncIterator[Connection]:
    """Async context manager for database connections."""
    conn = await connect(url)
    try:
        yield conn
    finally:
        await conn.close()

# Usage
async def process_data() -> None:
    async with database_connection("postgresql://...") as conn:
        await conn.execute("SELECT * FROM users")
```

**Rules:**
- **ALWAYS** use async context managers for async resources
- **NEVER** use regular context managers with async code
- **USE** `@asynccontextmanager` for async resource factories

---

## Logging

### Logging Best Practices

**Setup logging properly at application startup:**

```python
import logging
import sys
from pathlib import Path

def setup_logging(level: int = logging.INFO, json_format: bool = False) -> None:
    """Configure application logging."""
    if json_format:
        # Production: structured JSON logging
        import json
        from datetime import datetime
        
        class JSONFormatter(logging.Formatter):
            def format(self, record: logging.LogRecord) -> str:
                return json.dumps({
                    "timestamp": datetime.utcnow().isoformat(),
                    "level": record.levelname,
                    "logger": record.name,
                    "message": record.getMessage(),
                    "module": record.module,
                    "function": record.funcName,
                    "line": record.lineno,
                })
        
        formatter: logging.Formatter = JSONFormatter()
    else:
        # Development: human-readable
        formatter = logging.Formatter(
            "%(asctime)s | %(levelname)-8s | %(name)s:%(funcName)s:%(lineno)d - %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"
        )
    
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)
    
    logging.basicConfig(level=level, handlers=[handler], force=True)
    
    # Reduce noise from verbose libraries
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
```

### Structured Logging

**For production applications, use structured logging:**

**Comparison: structlog vs python-json-logger**

| Feature | structlog | python-json-logger |
|---------|-----------|-------------------|
| **Approach** | Full framework | JSON formatter only |
| **Complexity** | More features | Simple, lightweight |
| **Context Binding** | ✅ Built-in `.bind()` | ❌ Manual |
| **Development UI** | ✅ Pretty console | ❌ JSON only |
| **Best For** | New projects, microservices | Adding JSON to existing code |

**Recommendation:**
- **USE** structlog for new microservices and cloud-native apps
- **USE** python-json-logger for adding JSON to existing projects
- **ALWAYS** use structured logging in production
- **NEVER** rely on string formatting for searchable data

```python
# Option 1: structlog (recommended for new projects)
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()
logger.info("user_login", user_id=123, ip="1.2.3.4")

# Option 2: python-json-logger (for existing projects)
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter())
```

### Correlation IDs and Request Tracking

**Always track requests with correlation IDs:**

```python
from contextvars import ContextVar
import uuid

request_id_var: ContextVar[str] = ContextVar("request_id", default="")

async def handle_request() -> None:
    # Set correlation ID at request boundary
    request_id_var.set(str(uuid.uuid4()))
    
    logger.info("request_started", request_id=request_id_var.get())
    await process_request()

async def process_request() -> None:
    # Correlation ID available in all nested calls
    logger.info("processing", request_id=request_id_var.get())
```

### Log Levels - When to Use

- **DEBUG**: Detailed diagnostic info (variables, state)
- **INFO**: Normal operation events (request started, job completed)
- **WARNING**: Unexpected but handled situations (deprecated API used, fallback triggered)
- **ERROR**: Errors that prevent specific operations (failed to save file, API timeout)
- **CRITICAL**: System-wide failures (database down, out of memory)

**Rules:**
- **NEVER** log sensitive data (passwords, tokens, PII)
- **NEVER** use INFO for per-item processing in loops (use DEBUG)
- **ALWAYS** include context (user_id, request_id, resource_id)
- **ALWAYS** log exceptions with `logger.exception()` or `exc_info=True`

---

## Testing

### Test Framework
- **ALWAYS** use pytest
- **NEVER** use unittest for new projects
- Organize tests to mirror src/ structure

### Test Writing Rules
- **ALWAYS** use descriptive test names: `test_user_creation_with_valid_email`
- **ALWAYS** follow Arrange-Act-Assert pattern
- Use fixtures for setup/teardown
- Use parametrize for multiple test cases:
  ```python
  import pytest

  @pytest.mark.parametrize("input,expected", [
      (1, 2),
      (2, 4),
      (3, 6),
  ])
  def test_double(input: int, expected: int) -> None:
      assert double(input) == expected
  ```
- **ALWAYS** test edge cases and error conditions
- Aim for >80% code coverage, but prioritize meaningful tests

### Async Testing
- Use `pytest-asyncio` for async tests:
  ```python
  import pytest
  import httpx

  @pytest.mark.asyncio
  async def test_fetch_url():
      """Test async functions with pytest-asyncio."""
      result = await fetch_url("https://example.com")
      assert result is not None
      assert len(result) > 0

  @pytest.fixture
  async def async_client():
      """Fixture for async resources."""
      async with httpx.AsyncClient() as client:
          yield client
  ```

### Mocking in Tests
- Mock external dependencies to isolate tests:
  ```python
  from unittest.mock import AsyncMock, Mock, patch

  def test_with_mock():
      """Mock synchronous dependencies."""
      mock_db = Mock()
      mock_db.get_user.return_value = User(id=1, name="Alice")

      service = UserService(mock_db)
      user = service.fetch_user(1)

      assert user.name == "Alice"
      mock_db.get_user.assert_called_once_with(1)

  @pytest.mark.asyncio
  async def test_async_with_mock():
      """Mock async dependencies."""
      mock_client = AsyncMock()
      mock_response = AsyncMock()
      mock_response.text = "mocked response"
      mock_client.get.return_value = mock_response

      result = await fetch_url("https://example.com", mock_client)
      assert result == "mocked response"
      mock_client.get.assert_called_once()

  @patch("myapp.external_api.call")
  def test_with_patch(mock_call):
      """Patch external dependencies."""
      mock_call.return_value = {"status": "ok"}
      result = my_function()
      assert result["status"] == "ok"
  ```

### Coverage Configuration
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py"]

[tool.coverage.run]
source = ["src"]
branch = true
omit = ["*/tests/*", "*/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "def __str__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

### Test Organization
- Place tests in `tests/` directory
- Name test files `test_*.py`
- Use conftest.py for shared fixtures
- Group related tests in classes (optional)
- Mock external dependencies (APIs, databases, file systems)
- Use `@patch` for patching specific functions/methods
- Always test both success and failure paths

---

## Documentation

### Docstrings
- **ALWAYS** include docstrings for:
  - Public modules
  - Public classes
  - Public functions/methods
- **USE** Google or NumPy docstring style
- Include type hints in signatures, not docstrings
- Example:
  ```python
  def calculate_average(
      numbers: list[float],
      *,
      weights: list[float] | None = None
  ) -> float:
      """Calculate the average of a list of numbers.

      Args:
          numbers: List of numbers to average
          weights: Optional weights for weighted average

      Returns:
          The calculated average

      Raises:
          ValueError: If numbers is empty or weights length doesn't match

      Examples:
          >>> calculate_average([1, 2, 3])
          2.0
      """
  ```

### Comments
- **AVOID** obvious comments that repeat the code
- **USE** comments to explain WHY, not WHAT
- **USE** comments for complex algorithms or non-obvious logic
- **UPDATE** comments when code changes

### README
- **ALWAYS** include:
  - Project description
  - Installation instructions
  - Quick start example
  - Development setup
  - Testing instructions
- Keep README concise, link to full docs if needed

---

## Dependencies

### Dependency Management
- **PREFER** uv for new projects (fastest)
- **ACCEPTABLE**: Poetry or PDM
- **NEVER** use pip + requirements.txt for new projects
- **ALWAYS** specify version constraints:
  - Use `>=` for minimum version
  - Use `<` for known breaking versions
  - Example: `requests>=2.31.0,<3.0.0`

### Dependency Organization
```toml
[project]
dependencies = [
    "httpx>=0.25.0",
    "pydantic>=2.5.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.1.0",
    "ty>=0.1.0",
]
docs = [
    "mkdocs>=1.5.0",
    "mkdocs-material>=9.4.0",
]
```

### Common Modern Libraries
- **HTTP**: `httpx` (not requests for new async code)
- **Data validation**: `pydantic`
- **CLI**: `typer` or `click`
- **Datetime**: `pendulum` or built-in datetime with zoneinfo
- **Environment**: `pydantic-settings`
- **Testing**: `pytest`, `pytest-cov`, `pytest-asyncio`
- **Logging**: `structlog` (structured logging) or `python-json-logger` (JSON formatter)

---

## Code Quality Tools

### Required Tools
1. **Ruff** - formatting and linting
2. **Type checker** - mypy (default) or ty (faster alternative)
3. **pytest** - testing

### Ruff Configuration
```toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "ARG",    # flake8-unused-arguments
    "SIM",    # flake8-simplify
    "TCH",    # flake8-type-checking
    "PTH",    # flake8-use-pathlib
]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Type Checker Configuration

**Type Checker Comparison:**

| Feature | mypy | ty |
|---------|------|-----|
| **Status** | ✅ Stable, mature (1.0+) | ⚠️ Beta (0.0.5+) |
| **Speed** | Standard | 10-100x faster |
| **Language** | Python | Rust |
| **Ecosystem** | Extensive | Growing (very new) |
| **IDE Support** | Universal | VS Code, PyCharm, Neovim |
| **Production Ready** | ✅ Yes | ⚠️ Not yet - expect changes |
| **Best For** | Most projects, production | Experimentation, speed testing |
| **Maintainer** | Python community | Astral (Ruff/uv creators) |

**Recommendation:**
- **USE mypy** as default for all production projects
- **USE mypy** for existing projects and teams
- **CONSIDER ty** for new experimental projects or to test speed improvements
- **EXPECT** ty to evolve rapidly (breaking changes possible)

**mypy Configuration (Recommended default):**
```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_any_generics = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
```

**ty Configuration (Alternative):**
```toml
[tool.ty]
python_version = "3.12"
strict = true
# ty uses similar configuration to mypy
```

```bash
# Run ty with uvx (no installation)
uvx ty check

# Or install and run
uv add --dev ty
ty check
```

---

## Performance

### Memory Efficiency
- **ALWAYS** use `slots=True` for dataclasses with many instances
- **ALWAYS** use generators for large sequences
- Use `__slots__` for regular classes when creating many instances
- Prefer iterators over lists when possible

### Optimization Guidelines
- **FIRST** write clear, correct code
- **THEN** profile to find bottlenecks
- **NEVER** optimize prematurely
- Use `functools.lru_cache` or `functools.cache` for expensive pure functions
- Consider PyPy for CPU-bound workloads

---

## Security

### Input Validation
- **ALWAYS** validate external input (API, CLI, files)
- Use Pydantic models for structured data validation
- Sanitize user input before using in SQL, shell commands, or file paths
- **NEVER** use `eval()` or `exec()` on user input
- Example with Pydantic:
  ```python
  from pydantic import BaseModel, EmailStr, field_validator

  class UserInput(BaseModel):
      username: str
      email: EmailStr
      age: int

      @field_validator("username")
      @classmethod
      def validate_username(cls, v: str) -> str:
          if not v.isalnum():
              raise ValueError("Username must be alphanumeric")
          return v

      @field_validator("age")
      @classmethod
      def validate_age(cls, v: int) -> int:
          if not 0 < v < 150:
              raise ValueError("Invalid age")
          return v
  ```

### SQL Injection Prevention
- **ALWAYS** use parameterized queries:
  ```python
  import sqlite3

  # WRONG - vulnerable to SQL injection
  def get_user_wrong(username: str):
      conn = sqlite3.connect("db.sqlite")
      cursor = conn.cursor()
      cursor.execute(f"SELECT * FROM users WHERE username = '{username}'")  # ❌
      return cursor.fetchone()

  # CORRECT - use parameterized queries
  def get_user_safe(username: str):
      conn = sqlite3.connect("db.sqlite")
      cursor = conn.cursor()
      cursor.execute("SELECT * FROM users WHERE username = ?", (username,))  # ✅
      return cursor.fetchone()

  # With SQLAlchemy (even better)
  from sqlalchemy import select
  from sqlalchemy.orm import Session

  def get_user_sqlalchemy(session: Session, username: str):
      stmt = select(User).where(User.username == username)  # ✅ Safe by default
      return session.scalar(stmt)
  ```

### Path Traversal Prevention
- **ALWAYS** validate file paths:
  ```python
  from pathlib import Path

  def read_user_file_safe(filename: str, base_dir: Path) -> str:
      """Safely read user-provided filename."""
      # Resolve to absolute path and check it's within base_dir
      file_path = (base_dir / filename).resolve()

      if not file_path.is_relative_to(base_dir):
          raise ValueError("Invalid file path - outside allowed directory")

      if not file_path.exists():
          raise FileNotFoundError(f"File not found: {filename}")

      return file_path.read_text()
  ```

### Password Hashing
- **ALWAYS** use proper password hashing (Argon2 or bcrypt):
  ```python
  from argon2 import PasswordHasher
  from argon2.exceptions import VerifyMismatchError

  ph = PasswordHasher()

  def hash_password(password: str) -> str:
      """Hash password using Argon2."""
      return ph.hash(password)

  def verify_password(password: str, hash: str) -> bool:
      """Verify password against hash."""
      try:
          ph.verify(hash, password)
          return True
      except VerifyMismatchError:
          return False

  # Add to dependencies:
  # dependencies = ["argon2-cffi>=23.1.0"]
  ```

### Secrets Management
- **NEVER** hardcode secrets, API keys, or passwords
- Use environment variables or secret management systems
- Use `python-dotenv` or `pydantic-settings` for configuration
- **NEVER** commit `.env` files to version control
- Example with pydantic-settings:
  ```python
  from pydantic_settings import BaseSettings, SettingsConfigDict

  class Settings(BaseSettings):
      """Load configuration from environment variables."""
      model_config = SettingsConfigDict(
          env_file=".env",
          env_file_encoding="utf-8",
          case_sensitive=False,
      )

      database_url: str
      api_key: str
      secret_key: str

  # Usage
  settings = Settings()  # Loads from .env or environment

  # NEVER do this:
  # API_KEY = "hardcoded-key-123"  # ❌ WRONG
  ```

### Dependencies
- Regularly update dependencies for security patches
- Use `pip-audit` or similar tools to check for vulnerabilities
- Pin dependency versions in lock files

### Security Summary
- **ALWAYS** validate and sanitize user input
- **ALWAYS** use parameterized queries for SQL
- **ALWAYS** validate file paths against directory traversal
- **ALWAYS** use proper password hashing (Argon2, bcrypt)
- **NEVER** store secrets in code
- **ALWAYS** use HTTPS for external APIs
- Keep dependencies updated for security patches

---

## Anti-Patterns to Avoid

### Absolutely Forbidden
- **NEVER** use mutable default arguments: `def func(items=[])`
- **NEVER** modify a list while iterating over it
- **NEVER** use `import *`
- **NEVER** use bare `except:` (catches SystemExit, KeyboardInterrupt)
- **NEVER** silently swallow exceptions: `except Exception: pass`
- **NEVER** use eval or exec on untrusted input
- **NEVER** ignore type checker warnings without good reason
- **NEVER** commit print() statements for debugging (use logging)

**Catching `Exception` is acceptable when:**
- At top-level handlers (API endpoints, CLI entry points) - log and return error response
- In batch processing - log and continue to next item
- When wrapping with context: `raise NewError(...) from e`

### Discouraged Patterns
- Avoid god classes (classes that do too much)
- Avoid deep inheritance hierarchies (prefer composition)
- Avoid global state when possible
- Avoid circular imports
- Avoid monkey patching
- Avoid premature optimization
- Avoid clever one-liners that sacrifice readability

---

## Code Review Checklist

When reviewing code or writing new code, verify:

- [ ] Python 3.12+ features used appropriately
- [ ] All functions have type hints
- [ ] Modern type syntax used (`|`, lowercase generics, `type`, `collections.abc`)
- [ ] Advanced types where appropriate (Protocol, TypedDict, Literal)
- [ ] Dataclasses used for data containers with `slots=True`
- [ ] Pathlib used instead of os.path
- [ ] F-strings used for formatting
- [ ] Context managers used for resources
- [ ] Specific exceptions caught, not bare `except:`
- [ ] Async with proper error handling and timeouts
- [ ] Tests written for new functionality with mocking
- [ ] Async tests use pytest-asyncio
- [ ] Docstrings present for public APIs
- [ ] No hardcoded secrets or credentials
- [ ] Input validation and sanitization present
- [ ] No SQL injection vulnerabilities (parameterized queries)
- [ ] File paths validated against traversal
- [ ] Passwords properly hashed (if applicable)
- [ ] Ruff and type checker (mypy/ty) pass without errors
- [ ] No obvious security vulnerabilities
- [ ] Code is readable and maintainable

---

## Quick Reference

### Template for New Function
```python
def process_data[T](
    items: list[T],
    *,
    filter_func: Callable[[T], bool] | None = None,
    transform: bool = False,
) -> list[T]:
    """Process a list of items with optional filtering and transformation.

    Args:
        items: List of items to process
        filter_func: Optional filter function to apply
        transform: Whether to transform the items

    Returns:
        Processed list of items

    Raises:
        ValueError: If items is empty

    Examples:
        >>> process_data([1, 2, 3], filter_func=lambda x: x > 1)
        [2, 3]
    """
    if not items:
        raise ValueError("items cannot be empty")

    result = items
    if filter_func:
        result = [item for item in result if filter_func(item)]

    return result
```

### Template for New Class
```python
from dataclasses import dataclass
from typing import Self

@dataclass(frozen=True, slots=True)
class User:
    """Represents a user in the system.

    Attributes:
        id: Unique user identifier
        name: User's display name
        email: User's email address (optional)
    """
    id: int
    name: str
    email: str | None = None

    def with_email(self, email: str) -> Self:
        """Create a new User instance with updated email.

        Args:
            email: New email address

        Returns:
            New User instance with updated email
        """
        return type(self)(self.id, self.name, email)
```

---

## Python 3.13/3.14 Migration Notes

### Python 3.13 Changes

**Removed Modules:**
- `asyncore` and `asynchat` - **MIGRATE TO** `asyncio`
- `imp` - **MIGRATE TO** `importlib`
- Various deprecated stdlib modules ("dead batteries")

**Migration Example:**
```python
# Old (removed in 3.13)
import imp
imp.load_source('module', 'path/to/module.py')

# New (3.13+)
import importlib.util
spec = importlib.util.spec_from_file_location('module', 'path/to/module.py')
module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(module)
```

**New Features to Adopt:**
- `TypeIs` for better type narrowing (instead of `TypeGuard`)
- `ReadOnly` type hint for immutable structures
- Improved error messages (automatic, no action needed)

### Python 3.14 Changes

**Experimental Features (not for production):**
- JIT compiler: Enable with `PYTHON_JIT=1` environment variable
- Free-threaded mode (no-GIL): Enable with `PYTHON_GIL=0`

**Rules for 3.14:**
- **DO NOT** enable JIT or free-threading in production yet
- **DO** test experimental features in development/staging
- **EXPECT** breaking changes in experimental features
- **WAIT** for stable releases before production use

**Configuration Example:**
```toml
# pyproject.toml - Stay on 3.12 minimum for now
[project]
requires-python = ">=3.12"  # ✅ Safe
# requires-python = ">=3.14"  # ⚠️ Only if using experimental features
```

---

## Summary

These rules ensure modern, maintainable, and high-quality Python code. When in doubt:

1. **Prioritize clarity over cleverness**
2. **Let tools handle formatting and style**
3. **Type everything**
4. **Test thoroughly**
5. **Document public APIs**
6. **Keep it simple**

Follow these rules consistently, and your Python 3.12+ projects will be clean, performant, and maintainable.
