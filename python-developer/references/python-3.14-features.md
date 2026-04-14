# Python 3.14 Features

**Status:** Released (October 7, 2025)  
**Requires:** Python 3.14+

This document covers features and changes specific to Python 3.14. For core Python 3.12+ features, see [SKILL.md](../SKILL.md).

---

## Overview

Python 3.14 is the latest stable release with significant new features and improvements:

**Key Highlights:**
- **PEP 649/749**: Deferred evaluation of annotations
- **PEP 734**: Multiple interpreters in the standard library
- **PEP 750**: Template string literals (t-strings)
- **PEP 758**: Bracketless `except` expressions
- **PEP 768**: Safe external debugger interface
- **PEP 784**: Zstandard compression support
- Experimental JIT compiler (in official binaries)
- Free-threaded mode improvements (now officially supported)
- Incremental garbage collection
- Enhanced REPL with syntax highlighting
- Asyncio introspection capabilities

---

## Major New Features

### PEP 649 & 749: Deferred Evaluation of Annotations

Annotations are no longer evaluated eagerly. Instead, they're stored in special annotate functions and evaluated only when necessary.

**Benefits:**
- Improved performance (minimal runtime cost)
- No need to quote forward references
- Can introspect annotations at runtime

```python
from annotationlib import get_annotations, Format

def func(arg: UndefinedType) -> None:
    pass

# Different formats available
get_annotations(func, format=Format.VALUE)       # Evaluates to runtime values
get_annotations(func, format=Format.FORWARDREF)  # Returns ForwardRef objects
get_annotations(func, format=Format.STRING)      # Returns as strings
```

**Impact:**
- Most code continues working as-is
- Forward references no longer need quotes
- `from __future__ import annotations` may become unnecessary

### PEP 734: Multiple Interpreters in Standard Library

Run multiple isolated Python interpreters in the same process via the new `concurrent.interpreters` module.

**Benefits:**
- True multi-core parallelism (no GIL between interpreters)
- Isolation similar to multiprocessing but more efficient
- Human-friendly concurrency model (CSP, actor model)

```python
import concurrent.interpreters as interpreters

# Create and run code in isolated interpreter
interp = interpreters.create()
interpreters.run_string(interp, """
import sys
print(f"Running in interpreter {sys.version}")
""")
```

**Use Cases:**
- CPU-intensive parallel workloads
- Plugin systems with isolation
- Concurrent processing without multiprocessing overhead

**Current Limitations:**
- Starting interpreters not yet optimized
- Higher memory usage than ideal
- Limited sharing options between interpreters
- Some third-party extensions not yet compatible

Also added: `concurrent.futures.InterpreterPoolExecutor`

### PEP 750: Template String Literals (t-strings)

New template strings for custom string processing, similar to f-strings but return a `Template` object.

```python
from string.templatelib import Interpolation

# Create template with t-string
variety = 'Cheddar'
template = t'Try some {variety} cheese!'

# Access parts
for part in template:
    if isinstance(part, Interpolation):
        print(f"Interpolation: {part.value}")
    else:
        print(f"Static: {part}")

# Custom processing
def html_escape(template):
    """Example: escape HTML in templates."""
    parts = []
    for part in template:
        if isinstance(part, Interpolation):
            # Escape interpolated values
            parts.append(escape(str(part.value)))
        else:
            parts.append(part)
    return ''.join(parts)

user_input = '<script>alert("xss")</script>'
safe = html_escape(t'User said: {user_input}')
# Output: User said: &lt;script&gt;alert("xss")&lt;/script&gt;
```

**Use Cases:**
- SQL query builders with injection protection
- HTML/XML generation with automatic escaping
- Shell command builders
- Logging with structured data
- Domain-specific languages

### PEP 768: Safe External Debugger Interface

Zero-overhead debugging interface for attaching debuggers to running processes.

```python
import sys
from tempfile import NamedTemporaryFile

# Create debug script
with NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
    script_path = f.name
    f.write(f'import debugger; debugger.attach({os.getpid()})')

# Execute in process with PID 1234
sys.remote_exec(1234, script_path)
```

**Features:**
- No overhead at runtime
- Safe execution points
- Attach to production processes without stopping them

**Security Controls:**
- `PYTHON_DISABLE_REMOTE_DEBUG` environment variable
- `-X disable-remote-debug` command-line option
- `--without-remote-debug` configure flag

### PEP 784: Zstandard Compression Support

New `compression` package with Zstandard format support.

```python
from compression import zstd
import math

# Compress data
data = str(math.pi).encode() * 20
compressed = zstd.compress(data)
ratio = len(compressed) / len(data)
print(f"Compression ratio: {ratio:.2%}")

# Decompress
original = zstd.decompress(compressed)
```

**Also reorganized:**
- `compression.zstd` - New Zstandard support
- `compression.lzma` - Re-exports `lzma`
- `compression.bz2` - Re-exports `bz2`
- `compression.gzip` - Re-exports `gzip`
- `compression.zlib` - Re-exports `zlib`

**Integration:**
- `tarfile` supports Zstandard
- `zipfile` supports Zstandard
- `shutil` supports Zstandard archives

---

## Experimental JIT Compiler

### Overview

Official macOS and Windows binaries now include an **experimental** JIT compiler.

**Status:** Experimental - NOT for production use

### Enabling the JIT

```bash
# Set environment variable
PYTHON_JIT=1 python your_script.py

# Or for downstream builds
./configure --enable-experimental-jit=yes-off
```

### Expected Performance

- **Best case:** 20% faster for CPU-bound pure Python
- **Typical:** 10-20% improvement on `pyperformance` benchmarks
- **Range:** Can be 10% slower to 20% faster depending on workload

### What Works Well

```python
# JIT optimizes:
# - Numeric computations
# - Tight loops
# - Repeated function calls
# - CPU-bound operations

def cpu_intensive(n: int) -> int:
    total = 0
    for i in range(n):
        total += i * i
    return total
```

### Known Limitations

- Increased memory usage
- Longer startup time
- May not work with all C extensions
- Free-threaded builds don't support JIT
- No unwinding support for `gdb`/`perf` (Python profilers work)

### Introspection

```python
import sys

# Check if JIT is available
if sys._jit.is_available():
    print("JIT compiler is available")

# Check if JIT is enabled
if sys._jit.is_enabled():
    print("JIT is currently enabled")
```

---

## Free-threaded Python (No-GIL)

### Overview

Free-threaded mode (PEP 703) is now **officially supported** (no longer experimental).

**Status:** Officially supported, but still optional

### What Changed in 3.14

- Implementation completed (PEP 703)
- Specializing adaptive interpreter enabled (PEP 659)
- Performance penalty reduced to ~5-10% (single-threaded)
- More permanent solutions replace temporary workarounds

### Enabling Free-threading

```bash
# Use free-threaded build
python3.14t your_script.py

# Or set environment variable
PYTHON_GIL=0 python3.14 your_script.py
```

### Benefits

```python
from concurrent.futures import ThreadPoolExecutor

def cpu_task(n: int) -> int:
    """CPU-intensive task - truly parallel with no-GIL."""
    return sum(i * i for i in range(n))

# With no-GIL: uses multiple CPU cores effectively
with ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(cpu_task, [10_000_000] * 4))
```

### Thread Safety Requirements

```python
from threading import Lock

class Counter:
    def __init__(self) -> None:
        self._value = 0
        self._lock = Lock()
    
    def increment(self) -> None:
        # MUST use locks without GIL
        with self._lock:
            self._value += 1
```

### Windows Compatibility Note

On Windows with free-threaded builds, `Py_GIL_DISABLED` must now be explicitly defined by build backends.

### Context Inheritance

New `-X context_aware_warnings` flag (defaults to `True` in free-threaded builds):
- Enables concurrent-safe warnings control
- New `thread_inherit_context` flag
- Threads inherit `Context()` from caller

---

## Other Major Improvements

### Asyncio Introspection

New command-line tools for debugging async programs:

```bash
# List all asyncio tasks
python -m asyncio ps <PID>

# Show task tree
python -m asyncio pstree <PID>
```

**Example output:**
```
└── (T) Task-1
    └── main example.py:13
        ├── (T) worker-1
        │   └── process_data example.py:8
        └── (T) worker-2
            └── process_data example.py:8
```

**New functions:**
- `asyncio.capture_call_graph()`
- `asyncio.print_call_graph()`

### PEP 758: Bracketless `except` Expressions

Parentheses are now optional when catching multiple exception types, as long as no `as` clause is used.

```python
# Old — still valid
except (TimeoutError, ConnectionRefusedError):
    ...

# New — parentheses optional without `as`
except TimeoutError, ConnectionRefusedError:
    ...

# `as` still requires parentheses
except (TimeoutError, ConnectionRefusedError) as e:
    ...
```

**Key rules:**
- Both forms are semantically identical — purely a syntactic change
- Parentheses remain **required** when using `as`
- Fully backwards compatible — old parenthesized form continues to work
- Does **not** reintroduce Python 2 semantics: in Python 2 `except Foo, e:` bound the exception to a variable; that interpretation is gone

### Incremental Garbage Collection

The cycle GC is now incremental, reducing pause times by an order of magnitude for larger heaps.

**Changes:**
- Now only two generations: young and old
- `gc.collect(1)` performs an increment (not full generation 1)
- GC invoked less frequently when not called directly
- Collects young generation + increment of old generation

### Enhanced REPL

**Syntax Highlighting:**
- Enabled by default
- 4-bit VGA ANSI colors for compatibility
- Customizable via `_colorize.set_theme()` (experimental)
- Disable with `PYTHON_BASIC_REPL` environment variable

**Import Auto-completion:**
```python
# Type and press <Tab>
import co<Tab>       # Suggests: collections, codecs, contextlib, etc.
from pathlib import P<Tab>  # Suggests: Path, PosixPath, etc.
```

### Improved Error Messages

**Keyword typo suggestions:**
```python
>>> whille True:
...     pass
SyntaxError: invalid syntax. Did you mean 'while'?
```

**Better elif/else errors:**
```python
>>> if x:
...     pass
... else:
...     pass
... elif y:  # Error!
SyntaxError: 'elif' block follows an 'else' block
```

**Unhashable type errors:**
```python
>>> s = set()
>>> s.add({'key': 'value'})
TypeError: cannot use 'dict' as a set element (unhashable type: 'dict')
```

### A New Type of Interpreter

Experimental tail-call interpreter using small C functions instead of one large switch statement.

**Performance:** 3-5% faster on `pyperformance` (geometric mean)

**Requirements:**
- Clang 19+ on x86-64 or AArch64
- Enable with `--with-tail-call-interp`
- Use profile-guided optimization (PGO)

**Note:** This is an internal implementation detail, not user-facing.

---

## Standard Library Improvements

### New Modules

- `annotationlib` - Introspecting annotations (PEP 749)
- `compression` (including `compression.zstd`) - Compression modules (PEP 784)
- `concurrent.interpreters` - Multiple interpreters (PEP 734)
- `string.templatelib` - Template strings (PEP 750)

### Notable Module Updates

**argparse:**
- Color output for help text
- Suggestions for mistyped arguments/subparsers

**ast:**
- `ast.compare()` for comparing ASTs
- Support for `copy.replace()`

**asyncio:**
- Introspection with `capture_call_graph()` and `print_call_graph()`
- `create_task()` accepts arbitrary keyword arguments
- Free-threading support

**concurrent.futures:**
- `InterpreterPoolExecutor` for subinterpreters
- `ProcessPoolExecutor` now uses 'forkserver' by default on Unix (not macOS)

**json:**
- Command-line interface: `python -m json`
- Syntax highlighting in CLI output

**multiprocessing:**
- Default start method changed to 'forkserver' on Unix (not macOS)
- `SyncManager.set()` for shared sets

**pathlib:**
- `copy()`, `copy_into()`, `move()`, `move_into()` methods
- `info` attribute with `PathInfo` protocol

**pdb:**
- Remote attach: `python -m pdb -p <PID>`
- Syntax highlighting
- Auto-indent in multi-line input
- `pdb.set_trace_async()` for asyncio debugging

**pickle:**
- Default protocol now 5

**tarfile/zipfile:**
- Zstandard compression support

---

## Type System Enhancements

### Improved Type Narrowing

Better type inference and narrowing in 3.14:

```python
def process(value: int | str) -> None:
    if isinstance(value, int):
        # Type checker is smarter
        result = value * 2
    else:
        result = value.upper()
```

### Enhanced Protocol Support

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> str: ...
    def area(self) -> float: ...

# Better runtime protocol checking
shape: Drawable = Circle()
assert isinstance(shape, Drawable)  # Works better in 3.14
```

### Union Type Changes

`types.UnionType` and `typing.Union` are now aliases:

```python
# Both now produce the same type
Union[int, str]  # Old syntax
int | str         # New syntax

# Same repr for both
repr(Union[int, str])  # "int | str"
repr(int | str)        # "int | str"

# Can use in isinstance
isinstance(int | str, typing.Union)  # True
```

**Breaking changes:**
- Unions no longer cached
- Use `==` to compare unions, not `is`
- `__args__` attribute read-only
- Cannot set attributes on Union objects

---

## Performance Improvements

### General

- 5-10% reduction in pause times (incremental GC)
- Import time improved for many stdlib modules
- Faster dictionary, string, and function call operations
- Better attribute access performance

### Module-Specific

- **asyncio:** 10-20% faster (per-thread doubly linked list)
- **base64:** `b16decode()` up to 6x faster
- **io:** Opening/reading files 15% faster
- **pathlib:** `read_bytes()` 9-17% faster
- **uuid:** `uuid3()`/`uuid5()` 20-40% faster, `uuid4()` 30% faster

---

## Deprecations and Removals

### Removed in 3.14

**argparse:**
- `BooleanOptionalAction` parameters: `type`, `choices`, `metavar`

**ast:**
- `Bytes`, `Ellipsis`, `NameConstant`, `Num`, `Str` classes
- `Constant.n`, `Constant.s` properties

**asyncio:**
- Child watcher classes and functions
- `get_event_loop()` now raises `RuntimeError` (no implicit creation)

**email:**
- `localtime()` _isdst_ parameter

**importlib.abc:**
- `ResourceReader`, `Traversable`, `TraversableResources`

**itertools:**
- copy, deepcopy, pickle support

**pathlib:**
- Additional keyword arguments to `Path`
- Additional positional arguments to `relative_to()` and `is_relative_to()`

**sqlite3:**
- `version` and `version_info` (use `sqlite_version*`)
- Named placeholders with sequences

**urllib:**
- `Quoter`, `URLopener`, `FancyURLopener`

### New Deprecations

**asyncio:**
- `iscoroutinefunction()` - use `inspect.iscoroutinefunction()`
- Policy system (use `asyncio.run()` with `loop_factory`)

**codecs:**
- `codecs.open()` - use `open()` instead

**ctypes:**
- Setting `_pack_` on non-Windows (use `_layout_ = 'ms'`)

**os:**
- `os.popen()` and `os.spawn*` soft deprecated

**pathlib:**
- `PurePath.as_uri()` - use `Path.as_uri()`

---

## Migration Guide

### From Python 3.13 to 3.14

**Minimal Breaking Changes:**

Most code will work without modification. Key changes:

1. **Multiprocessing default changed:**
   - Unix (not macOS): 'forkserver' instead of 'fork'
   - May need to adjust for global state/pickling

2. **Annotations behavior:**
   - Mostly backwards compatible
   - May need `annotationlib` for advanced introspection

3. **Union types merged:**
   - Use `==` not `is` to compare unions
   - May affect runtime type introspection

4. **asyncio.get_event_loop():**
   - Now raises error - use `asyncio.run()` instead

### Testing Checklist

- [ ] Update Python version in configs
- [ ] Test without experimental features first
- [ ] Check multiprocessing/concurrent.futures for pickling issues
- [ ] Verify annotation-reading code still works
- [ ] Update asyncio code using `get_event_loop()`
- [ ] Test dependencies for 3.14 compatibility

---

## Configuration Updates

### pyproject.toml

```toml
[project]
requires-python = ">=3.14"

[tool.ruff]
target-version = "py314"

[tool.mypy]
python_version = "3.14"

[tool.ty]
python_version = "3.14"
```

### .python-version

```text
3.14
```

### Environment Variables

```bash
# Enable experimental JIT
PYTHON_JIT=1

# Enable free-threading (if using standard build)
PYTHON_GIL=0

# Disable remote debugging
PYTHON_DISABLE_REMOTE_DEBUG=1

# Disable syntax highlighting
PYTHON_BASIC_REPL=1
```

---

## When to Adopt Python 3.14

### ✅ Recommended For

- **New projects:** Take advantage of latest features
- **Development environments:** Test and learn new features
- **Non-critical applications:** After thorough testing
- **Projects needing Zstandard compression**
- **Projects needing template strings**

### ⚠️ Consider Carefully

- **Production systems:** Test extensively first
- **Projects with many dependencies:** Wait for ecosystem support
- **Critical infrastructure:** Use after community adoption

### 🚫 Avoid Experimental Features In Production

- **JIT compiler:** Still experimental
- **Free-threaded mode:** Officially supported but verify all dependencies
- **Remote debugging:** Understand security implications

---

## Resources

**Official Documentation:**
- [Python 3.14 Release Notes](https://docs.python.org/3.14/whatsnew/3.14.html)
- [Python 3.14 Documentation](https://docs.python.org/3.14/)

**PEPs:**
- [PEP 649](https://peps.python.org/pep-0649/) - Deferred Evaluation of Annotations
- [PEP 734](https://peps.python.org/pep-0734/) - Multiple Interpreters
- [PEP 741](https://peps.python.org/pep-0741/) - Python Configuration C API
- [PEP 744](https://peps.python.org/pep-0744/) - JIT Compiler
- [PEP 745](https://peps.python.org/pep-0745/) - Python 3.14 Release Schedule
- [PEP 750](https://peps.python.org/pep-0750/) - Template Strings
- [PEP 757](https://peps.python.org/pep-0757/) - Int Import/Export API
- [PEP 758](https://peps.python.org/pep-0758/) - Allow except without brackets
- [PEP 765](https://peps.python.org/pep-0765/) - Control flow in finally
- [PEP 768](https://peps.python.org/pep-0768/) - Safe External Debugger
- [PEP 776](https://peps.python.org/pep-0776/) - Emscripten tier 3
- [PEP 779](https://peps.python.org/pep-0779/) - Free-threaded Official Support
- [PEP 784](https://peps.python.org/pep-0784/) - Zstandard Compression

**Community:**
- [Python Discourse](https://discuss.python.org/)
- [Python Issue Tracker](https://github.com/python/cpython/issues)

---

## Summary

Python 3.14 represents a major milestone with:

**Production-Ready Features:**
- Deferred annotation evaluation (PEP 649/749)
- Multiple interpreters in stdlib (PEP 734)
- Template strings (PEP 750)
- Zstandard compression (PEP 784)
- Free-threaded mode officially supported (PEP 779)
- Incremental garbage collection
- Enhanced developer experience (REPL, error messages)

**Experimental Features:**
- JIT compiler (test only, not production)

**Key Takeaway:** Python 3.14 is production-ready with major new capabilities. The experimental JIT shows promise but needs more maturation. Free-threading is now officially supported and ready for broader adoption with compatible libraries.

**Release Date:** October 7, 2025  
**End of Life:** October 2030 (estimated)