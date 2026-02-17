# Core Language Features Reference Guide

This guide covers advanced Python language features including iterators, generators, decorators, descriptors, and metaclasses.

## Iterators and Generators

### Iterators

**Iterators** - Objects that implement `__iter__()` and `__next__()`

```python
from collections.abc import Iterator

class CountDown:
    """Custom iterator example."""
    def __init__(self, start: int) -> None:
        self.current = start

    def __iter__(self) -> Iterator[int]:
        return self

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# Usage
for num in CountDown(3):
    print(num)  # 3, 2, 1
```

### Generators

**Generators** - Functions that use `yield` to produce values lazily

```python
from collections.abc import Iterator

def fibonacci(n: int) -> Iterator[int]:
    """Generate first n Fibonacci numbers."""
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

# Memory efficient - only one value in memory at a time
for num in fibonacci(10):
    print(num)

# Generator expressions (like list comprehension but lazy)
squares = (x * x for x in range(1000000))  # No memory allocated yet
first_five = [next(squares) for _ in range(5)]  # Only compute what's needed
```

**When to use generators:**
- Processing large datasets that don't fit in memory
- Infinite sequences
- Pipeline processing with multiple transformations
- Reading large files line by line

```python
def read_large_file(path: Path) -> Iterator[str]:
    """Read file line by line without loading entire file."""
    with path.open() as f:
        for line in f:
            yield line.strip()

def process_logs(path: Path) -> Iterator[dict[str, str]]:
    """Pipeline: read -> filter -> transform."""
    for line in read_large_file(path):
        if "ERROR" in line:
            parts = line.split("|")
            yield {"timestamp": parts[0], "message": parts[1]}

# Chain generators efficiently
for log in process_logs("app.log"):
    print(log)
```

---

## Decorators

### Built-in Decorators

```python
from functools import wraps, cached_property
from typing import Any, Callable, TypeVar

class Circle:
    def __init__(self, radius: float) -> None:
        self._radius = radius

    @property
    def radius(self) -> float:
        """Read-only property."""
        return self._radius

    @property
    def area(self) -> float:
        """Computed property."""
        return 3.14159 * self._radius ** 2

    @cached_property
    def expensive_calc(self) -> float:
        """Cached property - computed once, then cached."""
        return self._radius ** 3 * 4.18879

    @staticmethod
    def from_diameter(diameter: float) -> "Circle":
        """Static method - no access to instance or class."""
        return Circle(diameter / 2)

    @classmethod
    def from_area(cls, area: float) -> "Circle":
        """Class method - receives class as first argument."""
        radius = (area / 3.14159) ** 0.5
        return cls(radius)
```

### Custom Decorators

```python
from functools import wraps
from typing import Any, Callable, TypeVar, ParamSpec
from time import time
import logging

P = ParamSpec("P")
T = TypeVar("T")

def timer(func: Callable[P, T]) -> Callable[P, T]:
    """Decorator to measure function execution time."""
    @wraps(func)  # Preserves function metadata
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        start = time()
        result = func(*args, **kwargs)
        elapsed = time() - start
        print(f"{func.__name__} took {elapsed:.2f}s")
        return result
    return wrapper

def retry(max_attempts: int = 3):
    """Decorator with arguments."""
    def decorator(func: Callable[P, T]) -> Callable[P, T]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    logging.warning(f"Attempt {attempt + 1} failed: {e}")
            raise RuntimeError("Unreachable")
        return wrapper
    return decorator

# Usage
@timer
@retry(max_attempts=3)
def fetch_data(url: str) -> dict[str, Any]:
    """Decorators stack from bottom to top."""
    import httpx
    return httpx.get(url).json()
```

### Async Decorators

```python
from functools import wraps
from typing import Callable, ParamSpec, TypeVar
import asyncio
from time import time

P = ParamSpec("P")
T = TypeVar("T")

def async_timer(func: Callable[P, T]) -> Callable[P, T]:
    """Decorator for async functions."""
    @wraps(func)
    async def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        start = time()
        result = await func(*args, **kwargs)
        elapsed = time() - start
        print(f"{func.__name__} took {elapsed:.2f}s")
        return result
    return wrapper

@async_timer
async def fetch_async(url: str) -> str:
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text
```

---

## Descriptors

**Descriptors** - Objects that implement `__get__`, `__set__`, or `__delete__` for attribute access control

### Custom Descriptors

```python
from typing import Any

class ValidatedString:
    """Descriptor for validated string attributes."""

    def __init__(self, min_length: int = 0, max_length: int = 100) -> None:
        self.min_length = min_length
        self.max_length = max_length

    def __set_name__(self, owner: type, name: str) -> None:
        """Called when descriptor is assigned to class attribute."""
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj: Any, objtype: type | None = None) -> str:
        """Called when attribute is accessed."""
        if obj is None:
            return self  # type: ignore
        return getattr(obj, self.private_name, "")

    def __set__(self, obj: Any, value: str) -> None:
        """Called when attribute is set."""
        if not isinstance(value, str):
            raise TypeError(f"{self.name} must be a string")
        if not self.min_length <= len(value) <= self.max_length:
            raise ValueError(
                f"{self.name} must be between {self.min_length} "
                f"and {self.max_length} characters"
            )
        setattr(obj, self.private_name, value)

class User:
    """Using descriptors for validation."""
    username = ValidatedString(min_length=3, max_length=20)
    email = ValidatedString(min_length=5, max_length=100)

    def __init__(self, username: str, email: str) -> None:
        self.username = username  # Triggers __set__
        self.email = email

# Usage
user = User("alice", "alice@example.com")
# user.username = "ab"  # Raises ValueError
```

### Property as Descriptor

```python
class Temperature:
    """Properties are descriptors under the hood."""

    def __init__(self, celsius: float) -> None:
        self._celsius = celsius

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError("Temperature below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        self.celsius = (value - 32) * 5/9
```

---

## Walrus Operator (`:=`)

**Assignment expressions** - Assign and use value in same expression (Python 3.8+)

```python
from pathlib import Path

# In if statements - avoid duplicate computation
def process_file(path: Path) -> str | None:
    if (content := path.read_text()) and len(content) > 0:
        return content.upper()
    return None

# In while loops - cleaner than infinite loop + break
def read_chunks(file_path: str, chunk_size: int = 1024) -> None:
    with open(file_path, "rb") as f:
        while (chunk := f.read(chunk_size)):
            process_chunk(chunk)

# In comprehensions - reuse computed value
data = [1, 2, 3, 4, 5]
if any((n := x * 2) > 5 for x in data):
    print(f"First value > 5: {n}")

# Filter and transform in one pass
results = [y for x in data if (y := x * 2) > 4]

# In match statements (Python 3.10+)
def process_command(cmd: str) -> None:
    match cmd.split():
        case ["load", path] if (p := Path(path)).exists():
            print(f"Loading from {p.absolute()}")
        case ["save", path] if (p := Path(path)).parent.exists():
            print(f"Saving to {p.absolute()}")
        case _:
            print("Unknown command")
```

**When to use walrus operator:**
- Avoid duplicate function calls or computations
- Make while loops more readable
- Combine filtering and transformation
- Capture values in comprehensions for later use

**When NOT to use:**
- Don't sacrifice readability for brevity
- Avoid in complex expressions where separate lines are clearer

---

## Context Variables

**Context variables** - Thread-safe state for async code (Python 3.7+)

```python
from contextvars import ContextVar
from collections.abc import Callable
from typing import TypeVar
import asyncio

# Define context variables at module level
request_id: ContextVar[str] = ContextVar("request_id", default="")
user_id: ContextVar[int | None] = ContextVar("user_id", default=None)

async def process_request(req_id: str, uid: int) -> None:
    """Each async task has its own context."""
    request_id.set(req_id)
    user_id.set(uid)

    await do_work()
    await do_more_work()

async def do_work() -> None:
    """Access context from anywhere in the call stack."""
    req_id = request_id.get()
    uid = user_id.get()
    print(f"Processing request {req_id} for user {uid}")

async def do_more_work() -> None:
    """Context is preserved across await points."""
    req_id = request_id.get()
    print(f"Still working on request {req_id}")

# Run multiple concurrent tasks - each has isolated context
async def main() -> None:
    await asyncio.gather(
        process_request("req-1", 100),
        process_request("req-2", 200),
        process_request("req-3", 300),
    )
```

### Practical Use Case - Logging

```python
from contextvars import ContextVar
import logging
from typing import Any

request_context: ContextVar[dict[str, Any]] = ContextVar(
    "request_context",
    default={}
)

class ContextualLogger:
    """Logger that includes context variables."""

    def __init__(self, logger: logging.Logger) -> None:
        self.logger = logger

    def _add_context(self, message: str) -> str:
        ctx = request_context.get()
        if ctx:
            context_str = " | ".join(f"{k}={v}" for k, v in ctx.items())
            return f"[{context_str}] {message}"
        return message

    def info(self, message: str) -> None:
        self.logger.info(self._add_context(message))

    def error(self, message: str) -> None:
        self.logger.error(self._add_context(message))

logger = ContextualLogger(logging.getLogger(__name__))

async def handle_request(request_id: str, user_id: int) -> None:
    """Set context for entire request."""
    request_context.set({"request_id": request_id, "user_id": user_id})

    logger.info("Starting request")  # Logs: [request_id=abc user_id=123] Starting request
    await process_data()
    logger.info("Request complete")

async def process_data() -> None:
    """Context automatically available in nested calls."""
    logger.info("Processing data")  # Context included automatically
```

**When to use context variables:**
- Request IDs in web applications
- User authentication context in async code
- Database transaction context
- Distributed tracing information
- Any state that should be task-local in async code

**Rules:**
- Define ContextVars at module level
- Use for task-local state in async code
- More reliable than thread-local in async environments
- Each async task gets isolated copy of context

---

## Pattern Matching - Advanced (Python 3.10+)

**Beyond basic matching:**

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

@dataclass
class Rectangle:
    top_left: Point
    bottom_right: Point

# Class patterns
def describe_shape(shape: Circle | Rectangle | Point) -> str:
    match shape:
        case Point(x=0, y=0):
            return "Origin point"
        case Point(x=0, y=y):
            return f"Point on Y axis at {y}"
        case Point(x=x, y=0):
            return f"Point on X axis at {x}"
        case Point(x=x, y=y):
            return f"Point at ({x}, {y})"
        case Circle(center=Point(x=0, y=0), radius=r):
            return f"Circle at origin with radius {r}"
        case Circle(radius=r) if r > 10:
            return f"Large circle with radius {r}"
        case Circle(center=center, radius=radius):
            return f"Circle at {center} with radius {radius}"
        case Rectangle(top_left=tl, bottom_right=br):
            width = br.x - tl.x
            height = br.y - tl.y
            return f"Rectangle {width}x{height}"

# Guards (if clauses)
def categorize_number(n: int) -> str:
    match n:
        case n if n < 0:
            return "negative"
        case 0:
            return "zero"
        case n if n % 2 == 0:
            return "positive even"
        case _:
            return "positive odd"

# Nested patterns
def process_config(config: dict[str, Any]) -> str:
    match config:
        case {"database": {"host": host, "port": port}}:
            return f"Database at {host}:{port}"
        case {"cache": {"enabled": True, "ttl": ttl}}:
            return f"Cache enabled with TTL {ttl}"
        case {"cache": {"enabled": False}}:
            return "Cache disabled"
        case _:
            return "Unknown configuration"

# Sequence patterns with *
def process_command(parts: list[str]) -> str:
    match parts:
        case []:
            return "Empty command"
        case ["help", *topics]:
            return f"Help for: {', '.join(topics) if topics else 'all'}"
        case ["install", package, *flags]:
            flag_str = ' '.join(flags) if flags else 'no flags'
            return f"Installing {package} with {flag_str}"
        case [cmd, *args]:
            return f"Unknown command: {cmd} with args: {args}"

# OR patterns
def is_exit_command(cmd: str) -> bool:
    match cmd.lower():
        case "quit" | "exit" | "q" | "bye":
            return True
        case _:
            return False

# AS patterns - capture matched value
def process_data(data: dict[str, Any]) -> None:
    match data:
        case {"result": {"value": value}} as full_result:
            print(f"Got value {value} from {full_result}")
        case {"error": error_msg} as error_data:
            print(f"Error: {error_msg}, full data: {error_data}")
```

**When to use pattern matching:**
- Complex conditional logic with structure unpacking
- API response handling with different shapes
- Command parsing
- Event handling with different event types
- State machine implementations

---

## Metaclasses (Advanced)

**Metaclasses** - Classes that create classes (rarely needed but good to know)

```python
from typing import Any

class SingletonMeta(type):
    """Metaclass for singleton pattern."""
    _instances: dict[type, Any] = {}

    def __call__(cls, *args: Any, **kwargs: Any) -> Any:
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    """Only one instance will ever exist."""
    def __init__(self, connection_string: str) -> None:
        self.connection_string = connection_string

# Both variables reference same instance
db1 = Database("postgresql://localhost")
db2 = Database("postgresql://other")  # Ignored, returns db1
assert db1 is db2

class ValidatedAttributesMeta(type):
    """Metaclass that adds validation to all string attributes."""

    def __new__(
        mcs,
        name: str,
        bases: tuple[type, ...],
        namespace: dict[str, Any]
    ) -> type:
        # Add automatic validation
        for key, value in namespace.items():
            if isinstance(value, str) and not key.startswith("_"):
                namespace[key] = value.strip()
        return super().__new__(mcs, name, bases, namespace)

class Config(metaclass=ValidatedAttributesMeta):
    database_host = "  localhost  "  # Automatically stripped
    api_key = "  secret  "  # Automatically stripped
```

**When to use metaclasses:**
- Framework development (ORMs, validation frameworks)
- Singleton patterns (though `@functools.cache` often better)
- Automatic registration of classes
- API design where you need to modify class creation

**When NOT to use:**
- Almost always - there's usually a simpler way
- Decorators, descriptors, or `__init_subclass__` are often better
- Rule: If you're not sure you need a metaclass, you don't need one