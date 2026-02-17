# Logging Best Practices Guide

Comprehensive guide to logging in Python applications, including setup, structured logging, async patterns, correlation tracking, and performance considerations.

## Basic Logging Setup

**Configuration:**
```python
import logging
import sys
from pathlib import Path

def setup_logging(
    level: int = logging.INFO,
    log_file: Path | None = None,
    json_format: bool = False
) -> None:
    """Configure logging for the application."""

    # Create formatter
    if json_format:
        # For structured logging in production
        import json
        from datetime import datetime

        class JSONFormatter(logging.Formatter):
            def format(self, record: logging.LogRecord) -> str:
                log_data = {
                    "timestamp": datetime.utcnow().isoformat(),
                    "level": record.levelname,
                    "logger": record.name,
                    "message": record.getMessage(),
                    "module": record.module,
                    "function": record.funcName,
                    "line": record.lineno,
                }
                if record.exc_info:
                    log_data["exception"] = self.formatException(record.exc_info)
                return json.dumps(log_data)

        formatter: logging.Formatter = JSONFormatter()
    else:
        # Human-readable format for development
        formatter = logging.Formatter(
            "%(asctime)s | %(levelname)-8s | %(name)s:%(funcName)s:%(lineno)d - %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"
        )

    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    console_handler.setLevel(level)

    # File handler with rotation (if specified)
    handlers: list[logging.Handler] = [console_handler]
    if log_file:
        from logging.handlers import RotatingFileHandler

        file_handler = RotatingFileHandler(
            log_file,
            maxBytes=10 * 1024 * 1024,  # 10MB
            backupCount=5,
            encoding="utf-8"
        )
        file_handler.setFormatter(formatter)
        file_handler.setLevel(level)
        handlers.append(file_handler)

    # Configure root logger
    logging.basicConfig(
        level=level,
        handlers=handlers,
        force=True  # Override any existing configuration
    )

    # Reduce noise from verbose libraries
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)

# Setup in application entry point
setup_logging(
    level=logging.INFO,
    log_file=Path("logs/app.log"),
    json_format=False  # Use True in production
)
```

---

## Structured Logging Libraries

**Comparison: structlog vs python-json-logger**

| Feature | structlog | python-json-logger |
|---------|-----------|-------------------|
| **Approach** | Full structured logging framework | JSON formatter for stdlib logging |
| **Complexity** | More complex, more features | Simple, lightweight |
| **Learning Curve** | Steeper | Minimal (just a formatter) |
| **Performance** | Fast, optimized | Fast, minimal overhead |
| **Context Binding** | ✅ Built-in `.bind()` | ❌ Manual via extra dict |
| **Context Variables** | ✅ Native support | ⚠️ Need custom filter |
| **Development UI** | ✅ Pretty console renderer | ❌ JSON only |
| **Processors** | ✅ Extensible pipeline | ❌ Limited |
| **Integration** | New API (`logger.info("event", key=val)`) | Standard logging API |
| **Ecosystem** | Good, growing | Excellent (stdlib compatible) |
| **Best For** | New projects, microservices, cloud-native | Existing projects, simple JSON needs |
| **Recommendation** | **Use for new projects** | Use for adding JSON to existing code |

### Option 1: structlog (Recommended for new projects)
```python
import sys
import structlog
from typing import Any

def configure_structlog(json_logs: bool = False) -> None:
    """Configure structlog with context preservation."""

    # Shared processors for all environments
    shared_processors = [
        structlog.contextvars.merge_contextvars,  # Add context variables
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.set_exc_info,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
    ]

    if json_logs:
        # Production: JSON output with structured tracebacks
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]
    else:
        # Development: Pretty console output
        # Automatically prints pretty tracebacks when "rich" is installed
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(),
        ]

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

# Auto-detect environment based on terminal
def configure_structlog_auto() -> None:
    """Auto-configure based on terminal detection."""
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
    ]

    if sys.stderr.isatty():
        # Pretty printing when we run in a terminal session.
        # Automatically prints pretty tracebacks when "rich" is installed
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(),
        ]
    else:
        # Print JSON when we run, e.g., in a Docker container.
        # Also print structured tracebacks.
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]

    structlog.configure(processors=processors)

# Get logger
logger = structlog.get_logger()

# Use with context - clean API
logger.info("user_login", user_id=123, ip_address="192.168.1.1")
logger.error("database_error", error="Connection timeout", retry_count=3)

# Bind context for multiple log calls
request_logger = logger.bind(request_id="req-123", user_id=456)
request_logger.info("processing_started")
request_logger.info("processing_completed", duration_ms=234)

# Add dependencies:
# dependencies = ["structlog>=23.2.0"]
```

### Option 2: python-json-logger (For existing stdlib code)
```python
import logging
from pythonjsonlogger import jsonlogger

def configure_json_logging() -> None:
    """Configure JSON logging with python-json-logger."""

    # Create JSON formatter
    formatter = jsonlogger.JsonFormatter(
        "%(asctime)s %(name)s %(levelname)s %(message)s",
        timestamp=True
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)

    logging.basicConfig(
        level=logging.INFO,
        handlers=[handler],
        force=True
    )

logger = logging.getLogger(__name__)

# Standard logging API - outputs JSON
logger.info("User login", extra={"user_id": 123, "ip_address": "192.168.1.1"})
logger.error("Database error", extra={"error": "Connection timeout", "retry_count": 3})

# Add dependencies:
# dependencies = ["python-json-logger>=2.0.7"]
```

**When to use which:**
- **Use structlog if:**
  - Starting a new project
  - Building microservices or cloud-native apps
  - Want clean structured logging API
  - Need context binding and processors
  - Want pretty dev console + JSON prod output

- **Use python-json-logger if:**
  - Adding JSON to existing codebase
  - Want minimal changes to current logging
  - Prefer standard logging API
  - Need simple JSON output only
  - Have large existing codebase with stdlib logging

---

## Log Levels - When to Use Each

```python
import logging

logger = logging.getLogger(__name__)

# DEBUG - Detailed diagnostic information
logger.debug("Processing record %s with config %s", record_id, config)
logger.debug("Cache hit for key: %s", cache_key)

# INFO - General informational messages
logger.info("Application started successfully")
logger.info("User %s logged in from %s", user_id, ip_address)
logger.info("Processing batch of %d items", len(items))

# WARNING - Something unexpected but not an error
logger.warning("API rate limit approaching: %d/%d", current, limit)
logger.warning("Deprecated feature used: %s", feature_name)
logger.warning("Retry attempt %d/%d for operation %s", attempt, max_attempts, operation)

# ERROR - Error that affects specific operation
logger.error("Failed to send email to %s: %s", email, error)
logger.error("Database query failed", exc_info=True)  # Include traceback

# CRITICAL - Severe error affecting entire application
logger.critical("Database connection pool exhausted")
logger.critical("Out of memory - shutting down")

# With exception info
try:
    risky_operation()
except Exception:
    logger.exception("Operation failed")  # Automatically includes traceback
```

**Rules for log levels:**
- **DEBUG**: Only in development, never in production by default
- **INFO**: Normal operations, major milestones
- **WARNING**: Recoverable issues, deprecated features
- **ERROR**: Operation failed but application continues
- **CRITICAL**: Application-level failures
- Use `.exception()` instead of `.error()` when logging caught exceptions

---

## Rotating File Handlers

```python
import logging
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler
from pathlib import Path

def setup_file_logging(log_dir: Path) -> None:
    """Setup file logging with rotation."""
    log_dir.mkdir(parents=True, exist_ok=True)

    # Size-based rotation
    size_handler = RotatingFileHandler(
        log_dir / "app.log",
        maxBytes=10 * 1024 * 1024,  # 10MB per file
        backupCount=5,               # Keep 5 backup files
        encoding="utf-8"
    )

    # Time-based rotation (daily)
    time_handler = TimedRotatingFileHandler(
        log_dir / "daily.log",
        when="midnight",        # Rotate at midnight
        interval=1,             # Every 1 day
        backupCount=30,         # Keep 30 days of logs
        encoding="utf-8"
    )

    # JSON logs for parsing/analysis
    json_handler = RotatingFileHandler(
        log_dir / "app.json",
        maxBytes=50 * 1024 * 1024,  # 50MB
        backupCount=10,
        encoding="utf-8"
    )

    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)s - %(message)s"
    )
    size_handler.setFormatter(formatter)
    time_handler.setFormatter(formatter)

    # Add handlers to root logger
    root_logger = logging.getLogger()
    root_logger.addHandler(size_handler)
    root_logger.addHandler(time_handler)
    root_logger.addHandler(json_handler)
```

---

## Logging in Async Code

```python
import logging
import asyncio
from contextvars import ContextVar
from typing import Any

logger = logging.getLogger(__name__)

# Context variable for request tracking
request_id_var: ContextVar[str] = ContextVar("request_id", default="")

class AsyncContextLogger:
    """Logger that includes async context in all messages."""

    def __init__(self, name: str) -> None:
        self.logger = logging.getLogger(name)

    def _get_extra(self) -> dict[str, Any]:
        """Get context variables as extra fields."""
        extra = {}
        if request_id := request_id_var.get():
            extra["request_id"] = request_id
        return extra

    def debug(self, msg: str, *args: Any, **kwargs: Any) -> None:
        kwargs.setdefault("extra", {}).update(self._get_extra())
        self.logger.debug(msg, *args, **kwargs)

    def info(self, msg: str, *args: Any, **kwargs: Any) -> None:
        kwargs.setdefault("extra", {}).update(self._get_extra())
        self.logger.info(msg, *args, **kwargs)

    def warning(self, msg: str, *args: Any, **kwargs: Any) -> None:
        kwargs.setdefault("extra", {}).update(self._get_extra())
        self.logger.warning(msg, *args, **kwargs)

    def error(self, msg: str, *args: Any, **kwargs: Any) -> None:
        kwargs.setdefault("extra", {}).update(self._get_extra())
        self.logger.error(msg, *args, **kwargs)

    def exception(self, msg: str, *args: Any, **kwargs: Any) -> None:
        kwargs.setdefault("extra", {}).update(self._get_extra())
        self.logger.exception(msg, *args, **kwargs)

# Usage
logger = AsyncContextLogger(__name__)

async def process_request(request_id: str) -> None:
    """Process request with context logging."""
    request_id_var.set(request_id)

    logger.info("Starting request processing")  # Includes request_id
    await do_async_work()
    logger.info("Request completed")  # Includes request_id

async def do_async_work() -> None:
    """Nested async function - context preserved."""
    logger.debug("Performing async operation")  # request_id still available
    await asyncio.sleep(0.1)

# Run concurrent tasks - each has isolated context
async def main() -> None:
    await asyncio.gather(
        process_request("req-001"),
        process_request("req-002"),
        process_request("req-003"),
    )
```

---

## Correlation IDs and Request Tracking

```python
import uuid
import logging
from contextvars import ContextVar
from typing import Any
from collections.abc import Callable, Awaitable
import asyncio

# Context variables for request tracking
request_id_var: ContextVar[str] = ContextVar("request_id", default="")
user_id_var: ContextVar[str] = ContextVar("user_id", default="")
correlation_id_var: ContextVar[str] = ContextVar("correlation_id", default="")

class CorrelationFilter(logging.Filter):
    """Add correlation data to all log records."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get() or "-"
        record.user_id = user_id_var.get() or "-"
        record.correlation_id = correlation_id_var.get() or "-"
        return True

def setup_correlation_logging() -> None:
    """Setup logging with correlation IDs."""
    formatter = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | "
        "[req:%(request_id)s|user:%(user_id)s|corr:%(correlation_id)s] | "
        "%(name)s - %(message)s"
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)
    handler.addFilter(CorrelationFilter())

    logging.basicConfig(level=logging.INFO, handlers=[handler], force=True)

# Middleware for request tracking
async def with_request_context(
    request_id: str | None = None,
    user_id: str | None = None,
    correlation_id: str | None = None,
) -> None:
    """Context manager to set request context."""
    request_id_var.set(request_id or str(uuid.uuid4()))
    if user_id:
        user_id_var.set(user_id)
    if correlation_id:
        correlation_id_var.set(correlation_id)

# Decorator for automatic request context
def with_correlation[T](
    func: Callable[..., Awaitable[T]]
) -> Callable[..., Awaitable[T]]:
    """Decorator to add correlation ID to async functions."""
    async def wrapper(*args: Any, **kwargs: Any) -> T:
        if not correlation_id_var.get():
            correlation_id_var.set(str(uuid.uuid4()))
        return await func(*args, **kwargs)
    return wrapper

logger = logging.getLogger(__name__)

# Usage example
@with_correlation
async def handle_api_request(user_id: str, data: dict[str, Any]) -> dict[str, Any]:
    """Handle API request with full tracing."""
    await with_request_context(
        request_id=str(uuid.uuid4()),
        user_id=user_id,
        correlation_id=correlation_id_var.get()
    )

    logger.info("Processing API request", extra={"data_keys": list(data.keys())})

    result = await process_data(data)

    logger.info("API request completed", extra={"result_size": len(result)})
    return result

async def process_data(data: dict[str, Any]) -> dict[str, Any]:
    """Process data - correlation context automatically available."""
    logger.debug("Starting data processing")

    # Call external service - pass correlation ID
    corr_id = correlation_id_var.get()
    await call_external_service(corr_id)

    logger.debug("Data processing complete")
    return {"status": "processed"}

async def call_external_service(correlation_id: str) -> None:
    """Call external service with correlation ID for distributed tracing."""
    logger.info(
        "Calling external service",
        extra={"service": "payment-api", "correlation_id": correlation_id}
    )
    # Send correlation_id in headers for distributed tracing
    # headers = {"X-Correlation-ID": correlation_id}
    await asyncio.sleep(0.1)
```

---

## Structured Logging with Context

```python
import structlog
from contextvars import ContextVar
from typing import Any

# Configure structlog with context
request_id_ctx: ContextVar[str] = ContextVar("request_id", default="")
user_id_ctx: ContextVar[str] = ContextVar("user_id", default="")

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

async def process_user_request(user_id: str, request_id: str) -> None:
    """Process request with structured logging."""
    # Bind context to logger for this request
    log = logger.bind(
        request_id=request_id,
        user_id=user_id,
        service="user-service"
    )

    log.info("request_started", endpoint="/api/users")

    try:
        await process_user(user_id)
        log.info("request_completed", status="success", duration_ms=123)
    except Exception as e:
        log.error(
            "request_failed",
            error=str(e),
            error_type=type(e).__name__,
            exc_info=True
        )
        raise

async def process_user(user_id: str) -> None:
    """Process user - context automatically included."""
    logger.info("processing_user", action="fetch_profile")
    # All logs include request_id and user_id from context
```

---

## Performance Considerations

```python
import logging
from typing import Any

logger = logging.getLogger(__name__)

# ❌ BAD - String concatenation happens even if not logged
logger.debug("Processing " + expensive_function())

# ✅ BETTER - Lazy string formatting (but function still called)
logger.debug("Processing %s", expensive_function())

# ✅ BEST - Check log level first for expensive operations
if logger.isEnabledFor(logging.DEBUG):
    expensive_data = compute_expensive_debug_data()
    logger.debug("Debug data: %s", expensive_data)

# ❌ BAD - f-string evaluated regardless of log level
logger.debug(f"User data: {fetch_user_data()}")

# ✅ GOOD - Use % or lazy evaluation
logger.debug("User data: %s", lambda: fetch_user_data())
```

---

## Logging Configuration Best Practices

**pyproject.toml configuration for logging dependencies:**
```toml
[project.optional-dependencies]
logging = [
    "structlog>=23.2.0",  # Recommended: Full structured logging framework
    # OR (not both):
    # "python-json-logger>=2.0.7",  # Alternative: Simple JSON formatter for stdlib
]
```

**Rules:**
- ALWAYS use `%s` formatting (lazy) not f-strings in log messages
- ALWAYS use `.exception()` when logging from except blocks
- NEVER log sensitive data (passwords, tokens, PII)
- Use DEBUG level sparingly - disable in production
- Use structured logging (structlog) for production systems
- Rotate log files to prevent disk space issues
- Include correlation IDs for distributed systems
- Use context variables for async-safe request tracking
- Filter out noisy third-party library logs
- Use JSON format in production, human-readable in development